---
title: 'Android 10属性系统原理,检测与定制源码反检测'
date: 2023-11-04
description: Android属性是Android系统中重要的设备与运行时信息的来源,也是改机和设备指纹收集绕不开的一个方向,本文就针对我对属性系统的理解,对属性改机的检测与反检测做一个总结.本文代码基于aosp 10.
lead: Android属性是Android系统中重要的设备与运行时信息的来源,也是改机和设备指纹收集绕不开的一个方向,本文就针对我对属性系统的理解,对属性改机的检测与反检测做一个总结.本文代码基于aosp 10.

tags:
  - "android"
categories:
  - "android"
sidebar: true #这个标签表示是否在文章页面显示侧边栏。在这个例子中，我们将其设置为 false，因为我们不想在这篇文章的页面上显示侧边栏。如果您想要在文章页面上显示侧边栏，您可以将其设置为 true。

pager: true #这个标签表示是否在文章列表页面上显示分页器。在这个例子中，我们将其设置为 false，因为我们不需要在文章列表页面上显示分页器。如果您有很多文章，并且想要将它们分成多个页面显示，您可以将其设置为 true。

weight: 1 #这个标签表示文章的权重。在 Hugo 中，权重决定了文章在列表中的排序顺序。如果您想要某篇文章排在列表的前面，您可以将其权重设置为更高的数字。在这个例子中，我们将其设置为 1，表示这篇文章应该排在列表的第一位。

#menu: main #这个标签表示文章所属的菜单。在 Hugo 中，您可以通过定义不同的菜单来组织您的网站导航。在这个例子中，我们将这篇文章添加到名为 "main" 的菜单中。如果您想要将文章添加到其他菜单中，您可以将其设置为相应的菜单名称。
#thumbnail: "img/placeholder.png" # Thumbnail image
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "post_toc"
---



# 一.Android属性原理:

## 属性系统基础:

属性由键值对组成,属性的值有类型,比如string,int,bool等,属性还关联着selinux上下文,因为属性是一种重要的资源,所以由selinux保护,由selinux的策略决定了哪些域可以读取/写哪些属性。

 

Android属性本质上是进程间通信的一种方式,由init进程创建`/dev/__properties__`目录,并在此目录下创建和属性相关的文件,然后init进程会调用带着MAP_SHARED标志的mmap()将这些文件映射到vm_area_struct,对属性的初始化和修改会反映到这些文件中去.其他需要读取和修改属性的进程也会调用带着MAP_SHARED标志的mmap()将这些文件映射到自己的vm_area_struct,因此本质上就是共享内存的方式。

 

在设计上基于安全的考虑,只有init进程可以初始化属性系统,即在`/dev/__properties__`目录下创建文件,也只有init进程可以修改属性,其他进程想修改属性需要通过`/dev/socket/property_service`这个socket文件向init进程发起通信请求,而且并不是所有进程都有权限访问这个socket文件,普通的app进程所在的untrusted_app域就是无法访问的,因此普通的app进程无法修改属性。

 

属性大致分为:
ro开头的只读属性,persist开头的持久化属性,ctl开头的控制属性,selinux.restorecon_recursive属性与普通属性。这么区分是因为init进程中对这些属性有不同的处理逻辑,后面会详细描述不同类型属性的差异。

 

基于效率的考虑,Android属性在属性文件格式的组织上经过了精心的设计,对于频繁的获取某一属性这种场景Android还提供了属性缓存的机制,这个机制也是改机的一个障碍。

 

进程间通信必然需要某种同步机制,Android中的属性通过包装futex系统调用实现了属性的同步功能。

 

系统提供了getprop工具可以获取所有属性值,调用的时候加上-Z参数可以获取属性对应的selinux上下文,加上-T参数可以获取属性值对应的类型。

## 属性核心API:

Android属性的实现核心代码放在了bionic libc中,有两个重要的头文件:`<sys/system_properties.h>`和`<sys/_system_properties.h>`描述了属性的api,这两个文件都处于路径`bionic/libc/include/sys/`下,这两个头文件不论是系统代码还是基于ndk开发的代码都是可以包含的,只不过包含`<sys/_system_properties.h>`文件的时候需要定义`"#define _REALLY_INCLUDE_SYS__SYSTEM_PROPERTIES_H_`",不然编译会报错,这是因为`<sys/_system_properties.h>`文件开头明确了:

```c
#ifndef _REALLY_INCLUDE_SYS__SYSTEM_PROPERTIES_H_
#error you should #include <sys/system_properties.h> instead
#endif
```

从这里也可以看出系统的开发者不想让你直接包含`<sys/_system_properties.h>`文件使用里边的接口,而最好使用`<sys/system_properties.h>`文件中的接口。

**`<sys/system_properties.h>`:**

这个文件包含的接口主要是设置属性,遍历属性以及读取属性的值,涵盖了正常使用属性系统的需求,其中有些接口需要prop_info结构体,这个结构体表示了属性值所在的区域.只不过调用接口的人无需关心prop_info结构体里边的细节,只需要调用一个接口获取prop_info指针,再将该指针传递给另外的接口获取更多信息。

 

**以下是接口描述:**

```c
int` `__system_property_set(const char``*` `__name, const char``*` `__value);
```

设置**name属性的值为**value,它的内部实现是通过`/dev/socket/property_service`这个socket文件向init进程发起通信请求,因此必须是selinux策略明确同意的进程才有权限发起通信,selinux策略同意设置属性的宏为set_prop,定义在te_macros文件中,这个宏其中之一的作用是允许指定的域connectto init进程域的unix_stream_socket文件.在system_app.te文件中指定了system_app可以设置的一些属性:

set_prop(system_app, bluetooth_audio_hal_prop)

set_prop(system_app, bluetooth_prop)

set_prop(system_app, debug_prop)

set_prop(system_app, system_prop)

表明system_app域中的进程既可以读取`u:object_r:bluetooth_audio_hal_prop:s0`上下文的属性值,也可以写这些上下文的属性值。

普通的app进程无法调用此函数,会报:
`type=1400 audit(0.0:901): avc: denied { write } for name="property_service" dev="tmpfs" ino=25226 scontext=u:r:untrusted_app:s0:c107,c256,c512,c768 tcontext=u:object_r:property_socket:s0 tclass=sock_file permissive=0 app=xxx.xxx.xxx`

```c
const prop_info* __system_property_find(const char* __name);
void __system_property_read_callback(const prop_info* __pi,void (*__callback)(void* __cookie, const char* __name, const char* __value, uint32_t __serial), void* __cookie)
```

这两个函数是一对,先调用`__system_property_find()`函数获取对应属性值的区域结构体指针prop_info*,然后调用`__system_property_read_callback()`以回调的方式接受属性值,属性值输出参数为**value,其中**cookie参数由调用方任意设置,最终调用**callback的时候会将**cookie再传递给回调函数。getprop程序获取单个属性值就是通过调用这两个函数实现的。

将获取属性拆分成两个函数是为了实现属性缓存机制,先将prop_info指针缓存起来,下次再获取属性值直接调用`__system_property_read_callback()`就行了,`bionic/libc/private/CachedProperty.h`提供了这种缓存机制,这种机制可以提高属性值获取效率。
其中__serial变量是用于标识属性值有没有变化的一个标识,它和属性的同步机制有关,后面会详细描述。

```
int` `__system_property_foreach(void (``*``__callback)(const prop_info``*` `__pi, void``*` `__cookie), void``*` `__cookie);
```

用于遍历所有属性值,进程只能遍历到自己有权限读取的属性值,对于进程有权限读取到的每一个属性都会回调一次**callback,在回调函数中通过得到的const prop_info\*** pi再调用`__system_property_read_callback()`函数就可以得到属性值.getprop程序获取所有的属性值就是通过这个函数实现的:

```c++
void PrintAllProperties(ResultType result_type) {
    std::vector<std::pair<std::string, std::string>> properties;
    __system_property_foreach(
        [](const prop_info* pi, void* cookie) {
            __system_property_read_callback(
                pi,
                [](void* cookie, const char* name, const char* value, unsigned) {
                    auto properties =
                        reinterpret_cast<std::vector<std::pair<std::string, std::string>>*>(cookie);
                    properties->emplace_back(name, value);
                },
                cookie);
        },
        &properties);
 
    std::sort(properties.begin(), properties.end());
```

```c
bool __system_property_wait(const prop_info* __pi, uint32_t __old_serial, uint32_t* __new_serial_ptr, const struct timespec* __relative_timeout)
```



用来等待参数const prop_info* **pi的serial成员不再为**old_serial值,变化以后的新值放在**new_serial_ptr参数中,并且可以指定**relative_timeout为等待的超时时间,__relative_timeout为null的时候表示永久等待.
这个接口可以用来实现这样的一个功能: 等待某个属性值变化成为一个新值,具体的代码可以看system/core/base/properties.cpp文件的WaitForProperty()函数。

### 下面是三个已经废弃的接口:

```c
int` `__system_property_read(const prop_info``*` `__pi, char``*` `__name, char``*` `__value);
```

和`__system_property_read_callback()`的作用差不多,只不过`__system_property_read_callback()`更灵活,所以这也可能是`__system_property_read`遭到废弃的原因。

```c
int` `__system_property_get(const char``*` `__name, char``*` `__value);
```

一步就可以获取__name参数对应的属性值,虽然此函数标记为废弃,但事实上Android框架中仍然在使用这个函数。

```c
const prop_info``*` `__system_property_find_nth(unsigned __n);
```

遍历所有的属性,得到第**n个属性对应的prop_info\*,**system_property_foreach完全可以替代这个函数,事实上Android框架中已经不再使用这个函数了。

### 再来看<sys/_system_properties.h>文件:

这个文件中的函数基本上都是系统在使用,所以一般不会使用到这个头文件:

```c
int` `__system_property_set_filename(const char``*` `__filename);
```

被废弃的函数,实现只是简单的返回-1。

```c
int` `__system_property_area_init(void);
```

专门给init进程调用的用于初始化属性区域的函数,调用此函数会以O_RDWR | O_CREAT标志打开`/dev/__properties__`目录下的文件,而这需要root权限。

```c
uint32_t __system_property_area_serial(void);
```

获取"global serial"用于实现缓存机制,上面介绍了一个比较常用的函数`__system_property_find`,它其实是一个比较耗时的函数,`__system_property_area_serial`函数的目的就是为了减少对`__system_property_find`函数的调用而将prop_info指针缓存起来,具体的代码在bionic/libc/private/CachedProperty.h文件中,"global serial"存放在`/dev/__properties__/properties_serial`文件中。

```c
int` `__system_property_add(const char``*` `__name, unsigned ``int` `__name_length, const char``*` `__value, unsigned ``int` `__value_length);
```

专门给init进程调用的用于添加属性的函数.因为调用这个函数需要root权限才能成功。

```c
int` `__system_property_update(prop_info``*` `__pi, const char``*` `__value, unsigned ``int` `__value_length);
```

专门给init进程调用的用于更新属性的函数.因为调用这个函数需要root权限才能成功。

```c
uint32_t __system_property_serial(const prop_info``*` `__pi);
```

返回某个属性区域的serial值,当prop_info的serial值发生了变化,就表示属性值发生了变化,此时可以重新调用`__system_property_read_callback`以获取更新后的属性值,代码同样见上面的`CachedProperty.h`文件的Get函数。

```c
int` `__system_properties_init(void);
```

由普通进程调用的初始化属性区域的函数,它的实现大致可以总结为就是open,mmap `/dev/__properties__`目录下的文件,只不过这些文件都是-r--r--r-- root root权限,因此普通进程只有读取的权限.需要使用属性系统的进程需要调用该函数。

```c
uint32_t __system_property_wait_any(uint32_t __old_serial);
```

废弃函数,由__system_property_wait取代。

## 基本原理:

这两个头文件定义的API讲完了,就可以总结一下属性的基本原理 :
init进程主要使用的是`<sys/_system_properties.h>`文件定义的接口,init进程在系统启动的时候负责创建`/dev/__properties__`目录下的文件,调用`__system_property_area_init`把文件映射到init进程的地址空间,并且通过`<sys/_system_properties.h`>定义的接口设置一些内置的属性,内置属性来源于系统编译时指定的属性文件以及内核命令行以及代码明确设置的属性.然后init开启socket等待接收其他进程设置属性的请求。

 

普通的进程主要使用的是`<sys/system_properties.h>`文件定义的接口,进程会在启动后调用`__system_properties_init`打开`/dev/__properties__`目录下的文件并映射到自己进程的地址空间(只读映射),然后通过`<sys/system_properties.h>`定义的接口来获取属性相关信息。

 

那么这两个头文件的实现在哪呢?实现由类SystemProperties表示,这个类定义在`bionic/libc/system_properties/include/system_properties/system_properties.h`文件中,创建这个类对象的文件位于`bionic/libc/bionic/system_property_api.cpp`,在这个文件中依次将上述接口的实现分发到SystemProperties类。

 

由于属性的头文件和定义都处于libc中,因此只要使用了libc.so的程序都可以使用这些接口,事实上普通进程的属性初始化动作`__system_properties_init`就是由libc自己调用的,`在bionic/libc/Android.bp`文件中，libc.so编译的时候依赖于静态库libc_init_dynamic,这个静态库的源码文件libc_init_dynamic.cpp中有这样的初始化语句:

```c++
__attribute__((constructor(``1``))) static void __libc_preinit() {
```

指明了so的初始化函数，这个函数最终在运行前由linker调用，在这个函数中会调用`__libc_init_common`，而在`__libc_init_common`函数中会最终调用`__system_properties_init`函数。

 

由于所有app进程都是由zygote fork并且"异化"出来的,zygote虚拟机启动的时候libc会调用`__system_properties_init`函数,因此所有java进程也同样的映射了`/dev/__properties__`目录下的文件,且映射区域的起始地址和大小和zygote一样,这一点可以通过`cat /proc/pid/maps | grep __prop | sort`得到印证。

 

如果阅读过linker的代码,会发现在linker_main函数中也会调用`__system_properties_init`函数,这个调用在libc调用之前,按照`__system_properties_init`函数实现的逻辑,调用两次会有问题.这是怎么回事呢?

 

事实上linker不可能再依赖于其他共享库,这个`__system_properties_init`函数最终会被扩展成`__dl___system_properties_init`,所以如果通过ida分析linker的话,会发现linker_main()中调用的`__dl___system_properties_init`,它初始化的是linker程序中.bss段的变量`__dl__ZL17system_properties`,因此这个调用和上面的libc中`__system_properties_init`的调用是不同的,并不会互相影响。添加前缀的逻辑在`linker/Android.bp`文件中的:

```c
// Insert an extra objcopy step to add prefix to symbols. This is needed to prevent gdb
// looking up symbols in the linker by mistake.
 prefix_symbols: "__dl_",
```

由于App进程有权限直接打开`/dev/__properties__`目录下的文件并且自己做解析,所以它并不一定非要使用`<sys/system_properties.h>`文件定义的接口,完全可以按照接口的实现自己写代码来实现属性操作,这也就可以绕开很多在接口层次做手脚的改属性机制。

 

对于系统层次的一些和属性相关的接口,基于上都是依靠`<sys/system_properties.h>和<sys/_system_properties.h>`实现的。

## 源码解析:

### init进程属性的初始化阶段:

下面分析一下属性系统的源码,从init进程开始,它主要负责创建和初始化属性区域.
查看`/dev/__properties__`目录下的文件,发现主要有三种文件:`property_info,properties_serial以及很多以selinux上下文命名的文件`

1. property_info文件表示所有属性以及它们的selinux上下文的对应关系:给定一个属性名称,可以通过这个文件查询出属性的selinux上下文,这个文件的映射区域由ContextsSerialized对象表示(依靠android::properties::PropertyInfoAreaFile),每个具体的上下文信息由ContextNode对象表示,ContextsSerialized对象中的ContextNode* context*nodes*数组就代表着所有的上下文对象信息。
2. properties_serial代表着"global serial",上面讲述接口的时候已经提及了,它由ContextsSerialized对象的prop_area* serial_prop*area*成员变量表示。
3. 以selinux上下文命名的文件,每个文件都聚集着具有相同selinux上下文的属性的信息,每个文件映射的区域由prop_area对象表示,每个具体的属性信息由prop_info对象表示。

总结起来就是SystemProperties对象成员变量为ContextsSerialized *contexts\*，contexts\*有成员变量ContextNode* context*nodes*表示每一个selinux上下文，每个ContextNode又有prop*area\* pa*成员变量表示对应的属性文件内容mmap的内存表示。

 

属性初始化函数在init进程的SecondStageMain函数中被调用,`init.cpp/SecondStageMain()`:

```c
void property_init() {
    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfo();
    if (__system_property_area_init()) {
        LOG(FATAL) << "Failed to initialize property area";
    }
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) << "Failed to load serialized property info file";
    }
}
```

在property_init函数中,首先调用CreateSerializedPropertyInfo(),它做的事情如下:

1. 读取以下文件,这些文件描述了属性和selinux上下文的对应关系:
   a) `/system/etc/selinux/plat_property_contexts`
   b)`/vendor/etc/selinux/vendor_property_context`s
   c) 如果`/product/etc/selinux/product_property_contexts`文件存在则也读取这个文件
   d) 如果`/odm/etc/selinux/odm_property_contexts`文件存在则也读取这个文件
   读取到的内容解析成PropertyInfoEntry数组,每个PropertyInfoEntry对象对应着属性的名称,selinux上下文,属性的类型以及是否为exact_match属性。
2. 调用BuildTrie函数将PropertyInfoEntry数组解析成多叉树结构(类似于目录的组织方式),树由TrieBuilder类表示,树的结点由TrieBuilderNode类表示,树的根结点对象为TrieBuilder类的成员变量TrieBuilderNode builder*root*
   TrieBuilder类另有两个成员变量std::set<std::string> contexts*和std::set<std::string> types*分别表示解析到的所有selinux上下文列表以及所有属性的类型列表。
   接下来将通过类TrieSerializer的SerializeTrie()函数来实现序列化,将TrieBuilder对象序列化为一个字符串,并且将字符串写入文件:`/dev/__properties__/property_info`,此文件本身的selinux上下文为:`u:object_r:property_info:s0`。
3. 调用`selinux_android_restorecon`将`/dev/__properties__/property_info`文件的selinux上下文设置为`u:object_r:property_info:s0`。

了解了CreateSerializedPropertyInfo()做的事情以后,来看一下上面操作的细节.
上面解析的文件指明了属性和selinux上下文对应关系,只不过有三种不同的情况,看文件内容示例:

1. `dalvik.vm.boot-dex2oat-threads u:object_r:exported_dalvik_prop:s0 exact int` , 这种是带有exact指示并且附带一个类型的。
2. `net.cdma u:object_r:net_radio_prop:s0`这种是一个属性名称加上一个selinux上下文的,属性名称不以.结尾。
3. `net. u:object_r:system_prop:s0` 这种也是一个属性名称加上一个selinux上下文的,只不过属性名称以.结尾 。

解析的时候会将信息放入结点,结点由TrieBuilderNode类表示,它有如下几个成员变量:

```c
PropertyEntryBuilder property_entry_;
std::vector<TrieBuilderNode> children_;
std::vector<PropertyEntryBuilder> prefixes_;
std::vector<PropertyEntryBuilder> exact_matches_;
```

builder*root*为根结点,它的属性值为root,selinux上下文为u:object_r:default_prop:s0。

 

当解析到`dalvik.vm.boot-dex2oat-threads`时,会形成如下的父子结点关系(每个结点都是TrieBuilderNode类型),子结点存放在父结点的children_列表中:
`root -> dalvik -> vm`,然后由于`dalvik.vm.boot-dex2oat-threads`带有exact指示,所以`boot-dex2oat-threads`成为了一个PropertyEntryBuilder对象存放在了vm结点的exact*matches*列表中。

 

当解析到`net.cdma`时,会形成如下的父子结点关系:
`root-> net`,由于`net.cdma`不以.结尾,所以它属于前缀属性,cdma成为了一个PropertyEntryBuilder对象存放在了net结点的prefixes_列表中。

 

当解析到`net.`时,会形成如下的父子结点关系:
`root -> net`,然后由于它以.结尾,所以net成为了一个PropertyEntryBuilder对象(上下文为`u:object_r:system_prop:s0`)存放于net结点自身的property*entry*成员变量。

 

按上面的规则把所有文件解析完毕以后,TrieBuilder类表示的树就形成了完整的结构。

 

接下来就调用类TrieSerializer的SerializeTrie()函数来实现将TrieBuilder序列化
序列化后的头由PropertyInfoAreaHeader结构体表示,紧接着PropertyInfoAreaHeader头结构后面的数据就是TrieBuilder序列化后的数据:

```c++
struct PropertyInfoAreaHeader {
  // The current version of this data as created by property service.
  uint32_t current_version; //值为1
  // The lowest version of libc that can properly parse this data.
  uint32_t minimum_supported_version; //值为1
  uint32_t size; //文件的大小
  uint32_t contexts_offset; //所有contexts字符串在文件中的偏移,偏移处内容为context的长度数组紧接着context字符串数组
  uint32_t types_offset; //所有contexts类型的字符串在文件中的偏移,偏移处内容为类型的长度数组紧接着类型字符串数组
  uint32_t root_offset; //TrieBuilder序列化后在文件中的偏移
};
```

序列化TrieBuilder的过程还是有点复杂的,位于`TrieSerializer::WriteTrieNode(const TrieBuilderNode& builder_node)`函数,序列化的过程中还会对children*,prefixes*和exact*matches*数组进行排序方便后面二分查找,其实我们知道反序列肯定可以还原得到类似于TrieBuilder结构就行了。

 

经过上面的步骤我们就可以生成了`/dev/__properties__/property_info`文件,如果想查找某个属性的值,首先需要通过这个文件查询出属性对应的selinux上下文,再通过和上下文同名的文件查询出属性的值。

 

下面我们再看一下`/dev/__properties__/property_info`文件的反序列化过程,并且看一下如何根据某个属性的值查询它的selinux上下文。

 

解析`/dev/__properties__/property_info`文件Android提供了帮助类,位于`system/core/property_service/libpropertyinfoparser`目录,其中property_info_parser.h头文件定义了和一些和反序列相关的类,其中PropertyInfoArea类就定义了一些读取出property_info文件信息的函数.看一下它的

```c
void PropertyInfoArea::GetPropertyInfo(const char``*` `property``, const char``*``*` `context,const char``*``*` `type``) const
```

调用GetPropertyInfo()函数就可以得到property属性对应的上下文和类型信息,分别做为context和type输出。

 

PropertyInfoArea::GetPropertyInfo主要通过调用`PropertyInfoArea::GetPropertyInfoIndexes`函数来得到属性的selinux上下文在所有上下文列表的下标,得到下标以后自然就可以得到上下文字符串了。

 

`PropertyInfoArea::GetPropertyInfoIndexes`函数逻辑如下:
首先出发点为树的root结点,然后将属性以.分割为不同的部分,顺着每个部分查找根结点下的子结点,再在子结点下查找下一级子结点,有点像给定一个路径从根路径一步步查找到包含该路径的目录的过程.由于子结点都是排过序的,因此查找都是二分查找.
一直查找到叶子结点(比如如果我们查询的是`ro.hardware.egl`,egl就为叶子结点的名称),在包含叶子结点的父结点的exact*matches*列表中匹配是否有元素和叶子结点名称相同,如果有则直接返回这个元素的下标.否则就在包含叶子结点的父结点的prefixes_列表中匹配是否有元素和叶子结点名称相同,如果还不能满足则以父结点的selinux上下文类型为准。

 

比如对上面的示例来说`net.abc`的selinux上下文就为`u:object_r:system_prop:s0`
`net.cdma.abc`的selinux上下文就为`u:object_r:net_radio_prop:s0`。

 

说完了CreateSerializedPropertyInfo,再回到init进程的property_init()函数,接下来调用的`__system_property_area_init`函数:

1. 调用`PropertyInfoAreaFile::LoadDefaultPath`会将上面的`/dev/__properties__/property_info`通过mmap映射到进程地址空间中,PropertyInfoAreaFile类的成员mmap*base*和mmap*size*分别代表着映射的基地址与映射大小，通过PropertyInfoAreaFile对象就可以查询每个属性的selinux上下文。

2. 调用`ContextsSerialized::InitializeContextNodes`,生成若干个ContextNode对象,长度为num_context_nodes,即代表了所有的property_info中的 selinux context。所有的ContextNode对象保存在context*nodes*成员变量中，这是一个ContextNode对象的指针数组。

3. 遍历上面的ContextNode对象指针数据,调用其中的函数 : context*nodes*[i].Open,这个函数会针对每个selinux上下文,,在`/dev/__properties__/`目录下生成相应的文件,比如`/dev/__properties__/u:object_r:apexd_prop:s0`,并且调用fsetxattr设置该文件的selinux上下文,selinx上下文就是文件名本身，并且调用mmap将该文件映射为一个prop_area对象.即表示文件内容为prop_area,prop_area的结构为:

   ```c
   uint32_t bytes_used_;
   atomic_uint_least32_t serial_;
   uint32_t magic_;
   uint32_t version_;
   uint32_t reserved_[28];
   char data_[0]
   ```

   prop_area映射的大小为PA_SIZE = 128 *1024，每一个属性的内容区域都是这么大，因此`ls -l /dev/__properties__/`看到里边文件的大小都为128* 1024。
   生成出来的prop*area对象就存放在ContextNode对象的pa*成员中。

4. 接下来会创建打开`/dev/__properties__/properties_serial`文件,同样的用fsetxattr设置该文件的上下文,并且调用mmap将该文件映射为一个prop_area对象。该对象保存于ContextsSerialized类的serial_prop*area*成员变量中。

接下来会调用`property_info_area.LoadDefaultPath()`,和上面的第1步执行的操作一样，只不过操作对象是init进程中的`static PropertyInfoAreaFile property_info_area`;

 

至此property_init()函数结束,属性区域的初始化就完成了,此时所有属性对应的上下文已经确定,并且属性文件已经创建出来,只不过此时属性文件是空的。

 

接下来init进程就会调用`uint32_t (*property_set)(const std::string& name, const std::string& value)`函数来设置属性了,属性的来源主要有如下几方面:

1. 内核命令行
2. 设备树文件
3. 环境变量
4. 代码直接设置,如ro.build.fingerprint,persist.sys.usb.config
5. 系统文件,如`/system/etc/prop.default,/system/build.prop,/vendor/default.prop`等

其中系统文件占据属性绝大部分的来源,它由编译时根据编译选项和各个产品的配置mk文件生成。

 

property_set函数最终会调用到`property_service.cpp`文件的PropertySet函数:

1. 调用`__system_property_find()`得到属性对应的prop_info*
2. 如果prop_info*不为空且属性以ro开头表明调用函数想更改已经设置了的ro属性,这是不允许的,因为ro属性是"write-once"
3. 如果prop_info*不为空且属性不以ro开头,则调用`__system_property_update`函数更新属性的值
4. 如果prop_info*为空说明属性不存在,则调用`__system_property_add`函数增加属性的值
5. 如果属性以persist开头则调用`WritePersistentProperty`函数将属性持久化
6. 调用property_changed()通知属性发生了变化

一开始属性是不存在的,因此会调用`__system_property_add`函数:

1. 调用contexts_的GetSerialPropArea函数得到prop_area* serial_pa,serial_pa代表着文件``/dev/__properties__/properties_serial`的内存映射,为"global serial"。
2. 调用contexts_->GetPropAreaForName(name)函数得到prop_area* pa,pa代表着属性名为name的selinux上下文同名文件的内存映射,init进程通过`open(filename, O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444)`打开该文件,因此如果这个文件不存在就会创建出来,然后调用`mmap(nullptr, pa_size_, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0)`将文件以读写共享的方式进行内存映射.这样init进程可以写该区域从而对属性值进行设置
3. 调用prop_area的add函数:`add(name, namelen, value, valuelen)`设置属性
4. 由于设置了新的属性,因此"global serial"发生了变化,值变为原来的值加1,此时调用__futex_wake通知等待在此serial处的进程,后边讲述属性同步机制的时候还会提及。

prop_area结构描述了具有同一个selinux上下文的所有属性区域,它的成员变量为:

```c
uint32_t bytes_used_;  //prop_area总共分配的字节大小
atomic_uint_least32_t serial_; //序列号，初始为0，当添加了新的属性或者属性值更新以后会自增1
uint32_t magic_; //0x504f5250
uint32_t version_; //0xfc6ed0ab
uint32_t reserved_[28]; //全为0的保留区域
char data_[0]; //属性区域，由二叉搜索树构成，树的每个结点类型为prop_bt
```

prop_area区域的属性区域由变种的字典树/二叉搜索树组成，树的每个结点类型为prop_bt:

```c
struct prop_bt {
  uint32_t namelen;  //名称长度
  atomic_uint_least32_t prop; //如果这个结点对应于一个属性，那么prop就是指向prop_area内存区域的prop_info结构的偏移
  atomic_uint_least32_t left; //左子树，其中所有结点的名称 < 此结点的名称
  atomic_uint_least32_t right;//右子树，其中所有结点的名称 > 此结点的名称
  atomic_uint_least32_t children;//下一级子结点
  char name[0];//名称所在的可变数组
 }
```

prop_bt代码上面的注释也说明了:
假设有ro.secure=1属性

![607812_G38DCBBEJCMEUSA](../../images/607812_G38DCBBEJCMEUSA.png)

1. ro属性先插入到树中，net和sys后插入，因为和ro同级，所以成为ro的左右子树。
2. ro.com属性的com节点和secure节点同级，它是后插入到树中的，因此com节点成为secure节点的左子树。
3. ro.secure属性值等于1，所以secure节点的prop指向prop_info结构代表着属性的名称和值。

再来看一下prop_info结构:

```c
struct prop_info {
    atomic_uint_least32_t serial;//修改属性值时用到的同步变量，同时也用来描述属性值的长度。
    union {
    char value[PROP_VALUE_MAX];//属性值字符串
    struct {
      char error_message[kLongLegacyErrorBufferSize];
      uint32_t offset;
    } long_property; //和长属性相关，属性值长度>=PROP_VALUE_MAX的为长属性
  };
  char name[0]; //属性名称字符串
}
```

再回到prop_area类的`bool add(const char* name, unsigned int namelen, const char* value, unsigned int valuelen)`函数，这个函数用来往prop_area中添加属性，也是二叉搜索树中的插入操作，理解这个操作以后遍历和更新属性也就好理解了，因为它们就是二叉搜索树的遍历和更新操作而已:
增加属性的prop_area::add函数调用的是find_property函数，查找属性的prop_area::find函数调用的也是find_property函数，两者主要区别在于最后一个参数bool alloc_if_needed，alloc_if_needed为true时，查找不到对应结点的时候会创建结点。

### prop_area::find_property函数分析：

1. 以.分割属性名称字符串，从树的根结点开始进入while(true)循环依次处理每一段分割开的子字符串。
2. 每一段循环都先查找当前处理结点的子结点，如果alloc_if_needed参数为true且没找到子结点，则调用new_prop_bt函数创建出一个子结点prop_bt对象。
3. 在子结点中查找当前循环处理的子字符串所对应的结点，如果这个子字符串等于此子结点的名称，那么就返回子结点做为下一次循环处理的结点，如果子字符串小于此子结点的名称，则返回子结点的左子树做为下一次循环处理的结点，如果子字符串大于此子结点的名称，则返回子结点的右子树做为下一次循环处理的结点。判断字符串大小逻辑为:先比较长度，长度小的就算小，长度相同的则取strncmp返回值。如果左右子树不存在且alloc_if_needed为true,同样的则调用new_prop_bt函数创建出左右子树的结点prop_bt对象。
4. 处理到最后一段子字符串时循环结束，如果是find查询操作则直接返回当前结点的prop指向的prop_info结构即可(有可能为null)，如果是add增加操作则调用new_prop_info函数添加一个新的prop_info结构并返回这个结构,属性值长度<<24放于prop_info结构的serial成员变量中。

回到init进程的SecondStageMain()函数，接下来调用StartPropertyService函数开启socket等待其他进程的设置属性请求:PROP_MSG_SETPROP和PROP_MSG_SETPROP2，至此App就可以使用属性系统了。

 

普通APP对于`/dev/__properties__/properties_serial`和`/dev/__properties__/property_info`都是有权限读取的，但是以selinx上下文命名的文件并不是所有的文件app都可以读取，这也就表示有的属性app是无法读取的。在app遍历属性的时候会调用`ContextNode::CheckAccessAndOpen`函数依次检查并打开所有的`/dev/__properties__`目录下的selinux上下文文件，对于selinux禁止读取的文件该函数返回false,app就遍历不到对应的属性。app有权限读取的属性在`system/sepolicy/public/domain.te`中指定,如:

```c
# Public readable properties
get_prop(domain, debug_prop)
get_prop(domain, exported_config_prop)
get_prop(domain, exported_default_prop)
get_prop(domain, exported_dumpstate_prop)
get_prop(domain, exported_fingerprint_prop)
get_prop(domain, exported_radio_prop)
get_prop(domain, exported_secure_prop)
get_prop(domain, exported_system_prop)
get_prop(domain, exported_vold_prop)
get_prop(domain, exported2_default_prop)
get_prop(domain, logd_prop)
```

## 属性分类:

上面提到了属性的几种分类，下面看一下分类的不同含义:

1. ro开头的只读属性，只能被设置一次

2. ctl开头的控制属性,它只能是如下的值:

   a) ctl.sigstop_on

   b) ctl.sigstop_off

   c) ctl.start

   d) ctl.stop

   e) ctl.restart

   f) ctl.interface_start

   g) ctl.interface_stop

   h) ctl.interface_restart

   

3. persist开头的持久化属性，属性值一般都是非持久化的，重启以后又会重新的初始化。而持久化属性在重启以后仍然会保留原来的值，持久化属性保存在`/data/property/persistent_properties`文件中。

4. persist开头的持久化属性，属性值一般都是非持久化的，重启以后又会重新的初始化。而持久化属性在重启以后仍然会保留原来的值，持久化属性保存在`/data/property/persistent_properties`文件中。

5. 其他属性。

## 属性同步:

由于属性是进程间共享的资源，而不同的进程可以读写属性，因此属性需要一种同步机制。属性同步依赖于futex系统调用,futex系统调用的第一个参数需要利用mmap机制在各个进程间共享，在属性实现中有如下几个变量提供这个参数:

1. `/dev/__properties__/properties_serial`文件内存映射成的prop*area结构的成员变量serial*。
2. 每个属性所对应prop_info结构的成员变量serial。

serial类型都为atomic_uint_least32_t，原子变量。

 

futex系统调用不仅可以实现多线程的事件通知机制，也可以实现互斥锁机制，bionic库封装了这个系统调用，比如**futex_wait,**futex_wake。

 

对于第1种情况`/dev/__properties__/properties_serial`文件中的serial*，初始值为0，当有属性被添加/更新时，serial*会自增1，并且通过**futex_wake唤醒所有等待着的进程，这些等待的进程是通过调用`**system_property*wait`函数(第一个参数为nullptr就可以等待这个serial*)从而调用`__futex_wait`进行等待的。由于属性被添加/更新时serial_会变化，所以`__system_property_wait`函数可以实现等待任意属性添加/更新的功能。serial_就是"global serial"。

 

对于第2种情况prop_info结构的成员变量serial初始值为属性值长度<<24,当更新了某个属性值的时候先将serial值|=1通知其他进程正在修改此属性，修改完属性以后将serial的值设置为(len << 24) | ((serial + 1) & 0xffffff)，然后通知其他所有等待serial值发生变化的进程。

 

假设一个进程正在修改某个属性，而另外一个进程要读取这个属性会发生什么？
看一下system_properties.cpp文件中SystemProperties::Get函数对同步的处理:

1. 首先调用prop_info *SystemProperties::Find(const char* name)函数根据名称查询prop_info对象。
2. 接着调用`int SystemProperties::Read(const prop_info* pi, char* name, char* value)`函数来获取属性的值，输入参数为value。在这个函数中首先进入一个循环，调用Serial函数等待serial值&1不为1，如果为1表示有进程正在修改此属性，此时等待修改进程更新完属性以后就读取到更新后的属性。如果读取进程判断没有进程修改，其他进程才开始修改，读取进程还会在读取属性以后再判断当前属性值是否发生了变化，如果发生了变化说明读取到的是旧属性值，它就重新进入循环再尝试读取新的属性值。所以如果有多个进程尝试读取更新属性值，读取进程有可能读取到旧属性，也有可能读取到更新后的属性。由于init进程通过socket接收更新属性的请求是串行的，所以不会有多个进程更新属性的情况。

# 二. 属性改机检测:

上面说完了属性的原理，下面谈一下我对属性改机的理解，属性改机最好是只针对某些app,只有这些app看到的属性是修改过的，不影响全局的属性，全局的属性如果修改了很可能系统工作就不正常了。且只针对某些app修改属性,修改属性的控制逻辑也比较好设计，可以更方便的实现一键改机，不需要重启机器。
先来看一下属性改机检测，主要分为两方面:

1. 检测属性实现机制有没有问题
2. 检测得到的属性值有没有可疑的比如ro.debuggable=1

这里说的主要是第一个方面:检测属性实现机制有没有问题，主要检测点有以下:

1. 检测属性实现所依赖的selinux是否正确开启
2. 多方面获取属性，检测获取到的属性值是否一致
3. 改属性会用到IO重定向，检测是否有IO重定向操作
   其他的检测比如是否root，是否有frida,xposed等等和属性检测没有直接关系就不再提及了。

先来看第1条检测selinux,厂商开启了selinux以后android系统部分一般都会遵守谷歌制定的策略，先调用getenforce判断selinux开启状态，然后去访问公开的被selinux策略禁止的文件，如果可以访问就表示selinux实现有问题，比如/dev/socket/property_service这个socket文件app是没办法open的，有一些属性文件selinux策略也是禁止访问的，系统中还有很多的文件可以用来做这种检测。读取/proc/self/mounts判断selinuxfs有没有挂载到/sys/fs/selinux。

### 第2条检测的逻辑如下:

1. 先调用getprop获取所有的属性,以此为基准对比其他途径获取的属性，一旦有差异基本上就代表着属性的实现有问题，为可疑设备。
2. 对比通过android.os.SystemProperties的get方法获取的属性，这个方法需要用反射来调用。
3. 对比android.os.Build类获取的属性，这个类在zygote preload class的时候就已经被初始化了，获取属性的时机很早，里边的属性有可能是原始的属性。
4. 对比通过jni调用的`__system_property_get`函数。
5. 对比通过jni调用的`__system_property_find`和`__system_property_read_callback`函数，引入这个两个函数需要加上`#if __ANDROID_API__ >= 26`。
6. 对比通过jni调用的遍历属性的函数`__system_property_foreach`和`__system_property_read_callback`，引入这个两个函数需要加上`#if __ANDROID_API__ >= 26`。
7. 对比调用"getprop+prop名称"获取的属性。
8. 自己解析`/system/etc/selinux/plat_property_contexts`等文件看获取到的属性上下文是否匹配。
9. 自己写代码解析`/dev/__properties__`下的文件，不使用libc的接口去获取属性并对比。

最后一种方式可以把系统和属性相关的代码拷贝到jni中，自己生成接口去获取属性，完全不依赖于系统的运行时。看下图:

 

![img](../../images/607812_ACC8NPPNC869NEG.png)

### 第3条的检测逻辑如下:

有很多改属性的逻辑都会将`/dev/__properties__`下的文件进行IO重定向，你读取`/dev/__properties__`下的文件，偷梁换柱让你读取到的是其他目录下的文件。IO重定向有很多种实现方式，这里检测其中的一些。

1. 有些IO重定向会直接修改内核`do_sys_open`函数，调用`do_filp_open`函数的时候传递另外一个filename过去，虽然可以实现IO重定向，但会在`/proc/pid/fd`下面暴露自己，打开文件的fd会指向被重定向的文件，检测手段:

   ```c++
   static std::string get_selfpath(int fd) {
    char path[PATH_MAX];
    snprintf(path, PATH_MAX, "%s/%d", "/proc/self/fd/", fd);
    
    char buff[PATH_MAX];
    ssize_t len = ::readlink(path, buff, sizeof(buff) - 1);
    if (len != -1) {
        buff[len] = '\0';
        return std::string(buff);
    }
    return "";
   }
   ```

   ```c++
   static bool checkFileFDLinkName(std::string filename) {
    int fd = open(filename.c_str(), O_RDONLY);
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "filename:%s,fd:%d", filename.c_str(), fd);
    std::string link_path = get_selfpath(fd);
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "link path: %s", link_path.c_str());
    return filename == link_path;
   }
   ```

   然后调用

   ```c++
   checkFileFDLinkName("/dev/__properties__/property_info")
           && checkFileFDLinkName("/proc/cpuinfo")
           && checkFileFDLinkName("/proc/meminfo")
           && checkFileFDLinkName("/dev/__properties__/u:object_r:default_prop:s0")
   ```

   

   逻辑是:存在这种IO重定向时，当你打开`/proc/cpuinfo`的时候，给你重定向到/xxx文件，假设`open("/proc/cpuinfo")`得到的fd为5，此时调用`readlink("/proc/pid/fd/5")`如果得到的路径是/xxx而不是`/proc/cpuinfo`，表示有问题。

2. 检测通过open和openat得到的文件内容是否相同，有些重定向判断某个文件是否需要重定向是通过检查内核里边do_sys_open的filename参数来实现的而忽略了dfd参数，此时将路径分隔开调用openat，如果得到的内容和调用open得到的内容不同则表明有问题。

   ```c++
   static bool checkOpenAt(std::string filename, std::string dir, std::string pathname) {
    int openfd = open(filename.c_str(), O_RDONLY);
    std::string open_content;
    ReadFdToString(openfd, &open_content);
    
    int dirfd = open(dir.c_str(), O_RDONLY);
    int openat_fd = openat(dirfd, pathname.c_str(), O_RDONLY);
    std::string openat_content;
    ReadFdToString(openat_fd, &openat_content);
    return open_content == openat_content;
   }
   ```

   

调用: checkOpenAt("/proc/cpuinfo", "/proc", "cpuinfo")

 

上面的检查可以很大程度上检测出属性是否被改机。

# 三.定制源码反检测:

那么如何实现自己的一套改属性的方案呢?
需要实现的目标如下:

1. 可以绕过上面的所有检测
2. 可以只针对某些app生效
3. 不需要重启手机
4. 不要改动原来的属性
5. 效率不能太差

在这里说一下我的实现方案:
要想绕过上面的所有检测，那么这个属性必须做的和真实的属性一样:selinux上下文是匹配的，属性区域也是存在并且匹配的，所有接口结果一致，除了里边内容是假的所有行为都和系统属性没什么差别。

 

先准备一份属性文件以及`/system/etc/selinux/plat_property_contexts，/vendor/etc/selinux/vendor_property_contexts`文件，这三个文件可以从真实的手机比如华为，oppo手机中获取。后面这两个文件需要先用脚本处理一下将厂商定义的selinux上下文修改为改机手机上已存在的selinux上下文，不然就得修改selinux策略文件添加这些上下文。

 

需要构建一个新的属性目录，假设为`/dev/xxx/__properties__`,init进程既初始化`/dev/__properties__`区域也初始化`/dev/xxx/__properties__`区域
`<sys/system_properties.h>和<sys/_system_properties.h>`头文件需要准备两份拷贝，我这里命名为`system_properties_ext.h和_system_properties_ext.h`,里边所有的函数都添加_ext后缀。

 

有了接口以后还需要准备一份接口实现文件`system_property_api_ext.cpp`,实现类为`static SystemProperties system_properties_ext;`这个类的实现依赖路径从`/dev/__properties__`修改为`/dev/xxx/__properties__`。
有了接口和实现类以后，init进程就可以利用新的接口初始化新区域:

```c++
#define EXT_ROOT_DIR  /dev/xxx
 
void property_init_ext() {
    mkdir(EXT_ROOT_DIR, S_IRWXU | S_IXGRP);
    mkdir(EXT_ROOT_DIR "/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfoExt();//读取的文件修改为上面准备好的plat_property_contexts和vendor_property_contexts文件
    if (__system_property_area_init_ext()) {
        LOG(FATAL) << "Failed to initialize property area";
    }
    if (!property_info_area_ext.LoadDefaultPathExt()) {
        LOG(FATAL) << "Failed to load serialized property info file";
    }
    init_ext_properties();
}
```

CreateSerializedPropertyInfoExt函数,`__system_property_area_init_ext`函数都已经切换为`/dev/xxx/__properties__`区域。

 

init_ext_properties函数用来设置属性的值，实现为解析上面准备的属性文件，调用InitPropertySetExt函数设置属性:

```c++
void init_ext_properties() {
    auto file_contents = std::string();
    const std::string filename = "/xxx/xxx/my_prop";
    if (!ReadFileToString(filename, &file_contents)) {
        PLOG(ERROR) << "Could not read properties from '" << filename << "'";
        return;
    }
    for (const auto& line : Split(file_contents, "\n")) {
        auto trimmed_line = Trim(line);
        if (trimmed_line.empty() || StartsWith(trimmed_line, "#")) {
            continue;
        }
 
        auto tokenizer = SpaceTokenizer(line);
 
        auto key = tokenizer.GetNext();
        auto value = tokenizer.GetRemaining();
        if (key.empty() || value.empty()) {
            continue;
        }
        InitPropertySetExt(key.substr(1, key.size() - 2), value.substr(1, value.size() - 2));
    }
}
```

property_init_ext函数调用以后新的区域`/dev/xxx/__properties__`中的属性就可以工作了。此时可以复制getprop程序，将其中所有的接口修改为加上_ext后缀，然后编译，调用这个程序如果能正确显示出改过的属性以及selinx上下文信息，表示新的属性区域是没有问题的。

 

接下来处理需要改机的app，上面说过zygote中已经调用过`__system_properties_init`初始化了属性区域，其他所有的app进程都是被zygote fork出来的，因此app继承了和zygote相同的属性区域，此时我们需要将app的属性区域切到新的属性区域`/dev/xxx/__properties__`下而且不能在mmap中留下痕迹。这里采用的机制是unshare+bind mount:
`com_android_internal_os_Zygote.cpp`文件的`SpecializeCommon`函数是fork一个新的app进程即将执行的"异化"函数,这个函数前仍然是以root权限运行，后面才放弃了权能，权限资源。我选择在这个函数动手，调用新的函数bind_mount_early:

```c++
static void bind_mount_early(uid_t uid, fail_fn_t fail_fn){
  if(getuid() == AID_WEBVIEW_ZYGOTE){
    return;
  }
  if (unshare(CLONE_NEWNS) == -1) {
    fail_fn(CREATE_ERROR("Failed to unshare(): %s", strerror(errno)));
  }
  if(!need_extra_prop(uid)){
    return;
  }
  // Create a second private mount namespace for our process
  if (TEMP_FAILURE_RETRY(mount("/dev/xxx/__properties__", "/dev/__properties__", nullptr,
                              MS_BIND | MS_REC, nullptr)) == -1) {
    fail_fn(CREATE_ERROR("Failed to mount /dev/xxx/__properties__ to /dev/__properties__: %s",
                         strerror(errno)));
  }
}
```

需要先判断是否为AID_WEBVIEW_ZYGOTE,webview zygote不需要处理。

 

调用`unshare(CLONE_NEWNS)`:正常的app会在下面的`MountEmulatedStorage`函数中调用`unshare(CLONE_NEWNS)`创建出新的mount namespace,这里我把unshare调用提前，为下面的bind mount做准备，将`/dev/xxx/__properties__` bind mount到`/dev/__properties__`。这样这个app的`/dev/xxx/__properties__`就会被重定向到`/dev/__properties__`,而且由于创建了新的mount namespace,所以只会对当前进程生效。

 

接着调用函数重新初始化属性区域:

```c++
static void handle_ext_prop(uid_t uid, fail_fn_t fail_fn){
  if(!need_ext_prop(uid)){
    return;
  }
  __system_properties_abandon();
  __system_properties_init();
 
}
```

`__system_properties_abandon`是我新添加的新函数为了释放掉之前属性区域的所有内存映射，接着调用`__system_properties_init`，这个函数是原先系统提供的函数，由于已经实现了bind mount，所以其实初始化的是`/dev/xxx/__properties__`区域，只不过对app来说它感知不到这点。

 

需要额外处理`android.os.Build`类，这个类初始化的比`SpecializeCommon`要早很多，我这里在ActivityThread类的`handleBindApplication方`法中处理:
判断如果当前进程需要修改属性，则用反射将Build类中的所有属性更新。

 

由于`bind mount`也会有特征: `/proc/pid/mountinfo,/proc/pid/mounts,/proc/pid/mountstats`中会显示出`/dev/__properties__`经过了Bind mount,因此需要修改内核去除该特征：
/proc文件系统中显示出这三个文件的逻辑最终都会走到`fs/seq_file.c`的seq_path_root函数中，其中`p = __d_path(path, root, buf, size)`返回的指针p就表示着目标bind点的字符串，只要修改这个字符串即可实现去除bind mount特征，比如判断如果p为`/dev/__properties__`，就修改p为其他字符串即可。

 

经过上面的调整基本上就实现了预期的目标，但是实际上还有遇到以下问题:

1. libhwui查找opengl库和gralloc库的时候会报错,因为这些实现依赖于原始的属性。

2. 涉及到CachedProperty类的时候会报错:

   `10``:``43``:``39.703` `6095` `6095` `F DEBUG  : signal ``11` `(SIGSEGV), code ``1` `(SEGV_MAPERR), fault addr ``0x79181ef108``10``:``43``:``39.769` `6095` `6095` `F DEBUG  : backtrace:``10``:``43``:``39.769` `6095` `6095` `F DEBUG  :    ``#00 pc 00000000000825e0 /apex/com.android.runtime/lib64/bionic/libc.so (SystemProperties::Serial(prop_info const*)+16) (BuildId: dd26adccaae5614ac299a5a8a88b6c69)``10``:``43``:``39.769` `6095` `6095` `F DEBUG  :    ``#01 pc 0000000000083c84 /apex/com.android.runtime/lib64/bionic/libc.so (bionic_trace_begin(char const*)+292) (BuildId: dd26adccaae5614ac299a5a8a88b6c69)``10``:``43``:``39.769` `6095` `6095` `F DEBUG  :    ``#02 pc 00000000000e1db8 /apex/com.android.runtime/lib64/bionic/libc.so (pthread_create+84) (BuildId: dd26adccaae5614ac299a5a8a88b6c69)`

第1条报错的原因是因为app的RenderThread线程使用了libhwui,libhwui依赖于opengl实现，opengl实现则依赖于原始的属性，它根据这些属性来查找厂商提供的opengl实现，具体的逻辑在系统的egl分发库EGL/Loader.cpp文件中。opengl实现还会依赖于gralloc实现，这个实现也是依赖于原始属性的。因此如果读取到了修改后的属性就会出错。

 

不论是Android还是Linux，都有一套opengl壳用来封装厂商提供的opengl/egl实现，壳会有自己的查找规则加载厂商的实现库。Android系统它的实现位于`frameworks/native/opengl/libs/EGL`目录，对于Linux来说使用的一般是`libglvnd: the GL Vendor-Neutral Dispatch library`。Android查找opengl实现用到的属性主要是:

ro.hardware.egl

ro.board.platform

而对于查找gralloc实现主要和以下几个属性有关:

ro.hardware.gralloc

ro.hardware

ro.product.board

ro.board.platform

ro.arch

### 解决方案:

RenderThread是系统在app启动过程中创建的线程，创建点为ActivityThread类的`handleLaunchActivity`函数中的`HardwareRenderer.preload();`调用。
如果可以让系统创建出的线程读取到的属性是原始属性，而App创建出来的线程读取到的是改过的属性就可以解决这个问题了。
pthread的线程局部存储机制`pthread_once,pthread_setspecific,pthread_getspecific`正好可以实现:
在`system_properties.h`文件创建接口:

```c
int` `__system_property_pthread_setspecific(const void ``*``value);``void``*` `__system_property_pthread_getspecific();
```

并创建`static pthread_key_t my_pthread_key`变量,在获取属性的时候判断调用者线程是否设置了线程局部存储，如果设置了就让调用者线程获取的所有属性都是原始的属性。获取原始属性又有两个方案:创建一个新的通道可以从init进程获取原始属性的值或者在`__system_properties_abandon`之前保存原始属性的值。这里我采用了后者，因为第一种方案selinux是不允许的，原则上尽量不去修改selinux策略。
然后在RenderThread启动以后调用`__system_property_pthread_setspecific`函数，这样就解决了第1个问题。

 

第2条报错原因是CachedProperty类实现了属性的缓冲机制，`__system_properties_abandon`释放掉之前属性区域的内存映射以后，CachedProperty还保存着之前的prop_info指针，解决方案就是判断如果已经释放了内存区域就清空CachedProperty类的`prop_info_，cached_area_serial_以及cached_property_serial_`成员。

经过上面对源码的修改，就可以通过之前所有的测试，且验机软件也会显示被修改后的属性:-)

ps:

1. 某大厂app会提前在别的线程中初始化egl,遇到这种报错直接在eglInitialize函数中调用`__system_property_pthread_setspecific`即可。
2. 设备指纹或者风控对设备的真实性是很多维度来判断，比如你改了内核，那我通过maps中的libc相关文件做check，记录签名，后端做备份，只要app在下线足够的多，那么判断还是能够做的
3. 杀软常规做法，收集白名单，只要一个文件不在白名单中，它就很可能是新的木马病毒。(文件列表白名单不够的，加载地址都获得了，直接dump段.text抽取特征不就完了。)
4. 对于大厂和安全乙方白名单，定制ROM都不能直接过去的，但是可以根据具体应用的检测方式做定制处理。说白了，安全是设备环境->系统环境->运行环境->网络环境->行为特征等一条供应链的安全咬合链条，任何一点崩坏，都有被入侵的风险。还有白名单的规则检测是基础的和直观的，模式是“发生->治理”；现在的检测都趋向与AI检测，模式是“预测->阻止”。所谓的上医治未病。



