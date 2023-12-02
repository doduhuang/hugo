---

title: "Root检测绕过技术"
date: "2023-12-2"
description: "通过FRIDA绕过检测"
categories:
  - "Android"
  - "FRIDA"
tags:
  - "Android"
  - "FRIDA"
menu: page # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
#thumbnail: "img/placeholder.png" # Thumbnail image
lead: "通过FRIDA绕过检测" # Lead text
comments: false # Enable Disqus comments for specific page
#authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "post_toc"
---

[TOC]

> #### 介绍
>
> 此文章翻译出处**[8ksed.io](https://8ksec.io/advanced-root-detection-bypass-techniques/)**
>
> Android设备上的Root检测相关的技术以及绕过它的方法。主要重点将放在应用开发者采用的策略上，以保护他们的应用程序并防止其在受损设备上运行。为了学习目的，我们将使用一个名为***Root Detector***的示例Root检测应用程序 [下载地址](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec-frida.apk)

示例应用程序已安装在我们的Root设备上，正如我们所看到的，它表明设备已经Root。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_roote_detector_app_shot1.png)



通过使用jadx-gui来反编译apk文件，以了解Root Detector应用程序在设备上安装后的操作。AndroidManifest.xml是所有Android应用程序的入口点，其中定义了应用程序的不同组件和服务。

## jadx-gui 查看XML Permissions 权限

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<permission android:name="com.8ksec.inappprotections.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION" android:protectionLevel="signature"/>
<uses-permission android:name="com.8ksec.inappprotections.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"/>
```

从权限的角度来看，似乎没有什么有趣的东西。主要使用存储权限。

除了这些定义的权限之外，看看这个应用程序中还有哪些其他组件。

```xml
<activity android:exported="true" android:name="com.8ksec.inappprotections.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
```

可以看到它只有一个活动，即MainActivity，它会被启动。

看看在这个MainActivity.java文件中有什么。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/advfrida-blog-img11.jpg)

可以看到jadx无法反编译大部分代码。这是因为代码混淆。但让我们尽量从我们已经反编译的代码中找出一些信息吧！

似乎MainActivity有一个静态块，在其中调用 `System.loadLibrary("inappprotections")` ，负责将本地库加载到内存中。

```java
static {
    String str = "ۧ۠ۧۤ۬۠ۗۛۙۡۦۚۙۡۧۢۛۡۘ۟ۛۥۢ۬۠۬ۨ۬۟ۤۙۙۢۘ۠ۡۛۤۨۨۨۙۜ۟۬ۜۜۙۦۘۤ۠ۨۨ۟ۡ";
    while (true) {
        switch ((((str.hashCode() ^ 436) ^ 234) ^ 912) ^ (-1907612566)) {
            case -1083933248:
                return;
            case 1567145872:
                System.loadLibrary("inappprotections");
                str = "۟ۚۨۘۤۜۧۘۛ۬ۧۖۤ۟ۨ۫ۜۘ۟ۢۥۜۦۦ۫۬ۖۘ۬ۤۢۙ۫ۥۘ";
                break;
        }
    }
}
```

由于它位于静态块中，这个本地库将在应用程序启动时加载。

除此之外，我们还可以观察到一个有趣的功能，名为 `detectRoot()`

```java
public final native int detectRoot();
```

通过查看函数的签名，它看起来是来自 `inappprotections` 库的本地函数，其返回值是整数。

通常，如果检测到根目录，这些函数会返回true，否则返回false。我们的目标是绕过根目录检测，以便在受损设备上运行该应用程序。

> 鉴于前提假设，我们可以使用Frida在Java层拦截函数。这将允许我们检查函数的返回值。如果返回值恰好是 `boolean` ，我们可以轻松地修改它，使其始终返回 `false` 。

## Frida Hooking Method

使用以下Frida脚本来挂钩 `detectRoot()` 函数

```js
let MainActivity = Java.use("com.8ksec.inappprotections.MainActivity");

MainActivity["detectRoot"].implementation = function () {
            let ret = this.detectRoot();
            console.log('detectRoot ret value is ' + ret);
            return ret;
};
```

这个钩子中，只是打印 `detectRoot()` 函数的返回值。使用frida运行这个脚本。

在运行frida之前，请确保frida-server已经在您的Android设备上运行。您可以使用类似FridaLoader（https://github.com/dineshshetty/FridaLoader/）的工具在您的root设备上运行frida服务器。

```bash
frida -U -l root_bypass.js -f com.8ksec.inappprotections
```

   *. . . .   Connected to Pixel 4a (id=0B151JEC202420)*
*Spawned `com.8ksec.inappprotections`. Resuming main thread!*
*[Pixel 4a::com.8ksec.inappprotections ]-> detectRoot is called*
*detectRoot ret value is 404*

在控制台中可以看到返回值是404。好的，所以它不仅仅是0或1。

再次运行脚本，看看我们是否得到相同的值：

*. . . .   Connected to Pixel 4a (id=0B151JEC202420)*
*Spawned `com.8ksec.inappprotections`. Resuming main thread!*
*[Pixel 4a::com.8ksec.inappprotections ]-> detectRoot is called*
*detectRoot ret value is 500*

这次我们得到了500，这是不一致的。不幸的是，仅仅操纵返回值可能不足以绕过这个特定的根检测情况。此外，代码是混淆的，仅仅基于这个类来理解根检测机制的基本逻辑是具有挑战性的。

因此，我们需要查看定义该函数的本地库，即这个 `inappprotections` 库。

## apktool提取APK

让我们快速使用apktool提取这个APK，这样我们就可以访问APK的内部文件和资源。

```bash
apktool d root_detector.apk
```

*I: Using Apktool 2.5.0-dirty on root_detector.apk**

**I: Loading resource table...*

*I: Decoding AndroidManifest.xml with resources...*

*I: Loading resource table from file: /home/kali/.local/share/apktool/framework/1.apk*

*I: Regular manifest package...*

*I: Decoding file-resources...*

*I: Decoding values / XMLs...*

*I: Baksmaling classes.dex...*

*I: Copying assets and libs...*

*I: Copying unknown files...*

*I: Copying original files...*

*I: Copying META-INF/services directory*

在lib文件夹中，我们可以找到 `libinappprotections.so` 库。

## Ghidra分析SO库

为了检查这个库，我们将使用Ghidra反汇编器。我们的目标是识别库中的导入函数，这些函数可以作为进一步分析可能有趣的函数的起点。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_imports.png)

看起来我们没有很多功能，我们能看到的第一个功能是我们的 `detectRoot` 功能。让我们快速导航到这个功能，查看反汇编并尝试理解逻辑。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_detectRoot_disassembly.png)

好的，所以这个函数本身并不是很大，但是通过快速查看这个反汇编代码，我们可以说它使用了间接分支来通过分析这个反汇编代码来打破控制流。X8寄存器在运行时动态加载，并且这个后续的分支指令将根据X8寄存器指向的地址调用函数。所以仅仅通过查看这个反汇编代码，我们无法预测将调用哪个函数。类似的间接分支指令在代码中随处可见。因此，仅仅通过对这个函数进行静态分析来理解其逻辑是相当困难的。



## Root Detection检测1

在继续之前，让我们对任何Android设备上通常如何执行root检测有一些了解。当我们对设备进行root操作时，它会在系统目录中放置一个名为 `su` 的可执行文件。为了检测 `su` 二进制文件，应用程序必须在默认路径（如system/bin/su）中检查 `su` 。这些路径必须在此库的某个地方硬编码。它很可能是硬编码在二进制文件的只读部分，因为所有硬编码的常量值都存在于该部分中。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_strings.png)

很遗憾，在字符串中没有找到与 `su` 路径相关的内容，我们可以看到文本部分存在许多随机字符。这表明字符串已被加密并存储在文本部分中。在这一点上我们无能为力。

### FRIDA hook libc.so fopen stat access

我们接下来应该考虑尝试什么？我们注意到有几个与文件处理相关的导入函数，包括access()、fopen()、fclose()和stat()。为了确认这一点，让我们对这些函数应用Frida拦截器。

```js
var arg0 = null;
Interceptor.attach(Module.findExportByName("libc.so", "fopen"), {
    onEnter: function (args) {
        arg0 = args[0];
        console.log(`fopen: ${args[0].readCString()}`);
    }
})

Interceptor.attach(Module.findExportByName("libc.so", "stat"), {
    onEnter: function (args) {
        console.log(`stat: ${args[0].readCString()}`);
    }
})

Interceptor.attach(Module.findExportByName("libc.so", "access"), {
    onEnter: function (args) {
        console.log(`access: ${args[0].readCString()}`);
    }
})
```

让我们现在运行脚本，看看我们得到了什么：

*. . . .   Connected to Pixel 4a (id=0B151JEC202420)*

*Spawned `com.8ksec.inappprotections`. Resuming main thread!*

*[Pixel 4a::com.8ksec.inappprotections ]-> stat: /data/app/~~MxHO_xqdQgeBWi4_Wpoj4A==/com.8ksec.inappprotections-1XZhJp18cue2EZWsYq4xjA==/base.apk*

*stat: /data/app/~~MxHO_xqdQgeBWi4_Wpoj4A==/com.8ksec.inappprotections-1XZhJp18cue2EZWsYq4xjA==/base.apk*

*stat: /data*

*stat: /data*

*stat: /data/dalvik-cache/arm64*

*access: /data/app/~~MxHO_xqdQgeBWi4_Wpoj4A==/com.8ksec.inappprotections-1XZhJp18cue2EZWsYq4xjA==*



### FRIDA hook linker64

我们目前正在接收大量的输出，其中很多似乎与应用程序在执行过程中访问的默认路径有关。为了改进我们的结果并获得更有意义的输出，我们应该在成功加载我们的libinappprotections库之后才调用这些钩子。为了实现这一点，我们将钩入linker64，因为它在最初将库加载到内存中起着关键作用。

```js
var do_dlopen = null;
var call_constructor = null;
Process.findModuleByName('linker64').enumerateSymbols().forEach(function (symbol) {
    if (symbol.name.indexOf('do_dlopen') >= 0) {
        do_dlopen = symbol.address;
    } else if (symbol.name.indexOf('call_constructor') >= 0) {
        call_constructor = symbol.address;
    }
})
```

我们可以通过使用frida的Process.findModuleByName() API轻松找到linker64模块，然后我们需要枚举linker64中的所有符号，以获取do_dlopen()和call_constructor函数的地址。当linker64尝试将库加载到内存中时，将调用这些函数。

一旦我们获得了这些地址，我们就可以附加钩子并跟踪链接器加载的所有库。

```js
var lib_loaded = 0;
Interceptor.attach(do_dlopen, function () {
    var library_path = this.context.x0.readCString();
    if (library_path.indexOf('libnative-lib.so') >= 0) {
        console.log(`Target library is loading...`);

        Interceptor.attach(call_constructor, function () {
            if (lib_loaded == 0) {
                var native_mod = Process.findModuleByName('libinappprotections.so');
                console.log(`Target library loaded at ${native_mod.base}`);

            }
            lib_loaded = 1;
        })
    }
})
```

让我们再次运行脚本并查看拦截到的数据

```bash
frida -U -l root_bypass.js -f com.8ksec.inappprotections
```


   *. . . .   Connected to Pixel 4a (id=0B151JEC202420)*
*Spawned `com.8ksec.inappprotections`. Resuming main thread!*
*[Pixel 4a::com.8ksec.inappprotections ]-> Target library is loading...*
*Target library loaded at 0x6d5d956000*
*stat: /data/app/~~MxHO_xqdQgeBWi4_Wpoj4A==/com.8ksec.inappprotections-1XZhJp18cue2EZWsYq4xjA==/base.apk*
*stat: /data/resource-cache/product@overlay@NavigationBarModeGestural@NavigationBarModeGesturalOverlay.apk@idmap*
*stat: /product/overlay/NavigationBarModeGestural/NavigationBarModeGesturalOverlay.apk*
*access: /data/user/0/com.8ksec.inappprotections*
*access: /data/user/0/com.8ksec.inappprotections/cache*
*access: /data/user_de/0/com.8ksec.inappprotections*
*access: /data/user_de/0/com.8ksec.inappprotections/code_cache*
*stat: /system/framework/framework-res.apk*
*stat: /data/resource-cache/product@overlay@GoogleConfigOverlay.apk@idmap*
*stat: /product/overlay/GoogleConfigOverlay.apk*
*stat: /data/resource-cache/product@overlay@GoogleWebViewOverlay.apk@idmap*
*stat: /product/overlay/GoogleWebViewOverlay.apk*
*stat: /data/resource-cache/product@overlay@NavigationBarModeGestural@NavigationBarModeGesturalOverlay.apk@idmap*
*stat: /product/overlay/NavigationBarModeGestural/NavigationBarModeGesturalOverlay.apk*
*access: /system/xbin/su*
*access: /system/bin/su*
*detectRoot ret value is 669*
*access: /dev/hwbinder*
*stat: /vendor/lib64/hw/gralloc.sm6150.so*

我们可以观察到我们的 `detectRoot()` 函数的执行。函数被调用，随后应用程序尝试使用access()函数访问某些系统路径。重要的是要注意，这些特定路径将存在于任何已root的设备上。在访问路径之后，函数根据一个常量值来确定是否检测到了root访问权限。

现在，让我们探索绕过这个根检测机制的潜在策略。

### FRIDA 修改函数 access 参数

一种可行的方法是通过操纵access()函数检查的路径，并提供一个虚假的路径。这种修改会让应用程序误以为`su`二进制路径不存在，从而导致返回非root状态。为了实施这个解决方法，让我们相应地修改脚本。

```js
Interceptor.attach(Module.findExportByName("libc.so", "access"), {
    onEnter: function (args) {
        var path = args[0].readCString();
        if(path.indexOf("/su") >= 0){
            console.log(`Manipulating su path...`);
            args[0].writeUtf8String("/system/nonexisting");
        }
        console.log(`access: ${args[0].readCString()}`);
    }
})
```

**再次运行脚本。**

*Manipulating su path...*
*access: /system/nonexisting*
*access: ing*
*Manipulating su path...*
*access: /system/nonexisting*
*access: onexisting*
*Manipulating su path...*
*access: /system/nonexisting*
*fopen:*
*stat: /sys/fs/selinux/class/security/index*
*detectRoot ret value is 572*

这次我们可以观察到，访问函数现在试图访问我们修改后的不存在的路径，而不是 `su` 路径。但是应用程序的用户界面仍然显示检测到了根设备。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_roote_detector_app_shot1-17015226149436.png)

这可能是因为该应用程序可能正在搜索与根检测相关的其他工件。

## Root Detection 检测 2

### FRIDA hook stat 修改参数

回到输出日志，我们可以观察到在返回值之前，它调用了一个 `stat()` 函数，该函数试图访问 `selinux policies` 文件。我们可以假设该应用程序正在尝试访问这些 `selinux policy` 以检测root权限。为了再次绕过此检查，我们可以使用相同的方法来修改传递给 `stat` 函数的输入参数，使用frida：

```js
Interceptor.attach(Module.findExportByName("libc.so", "stat"), {
    onEnter: function (args) {
        var path = args[0].readCString();
        if(path.indexOf("/selinux") >= 0){
            console.log(`Manipulating selinux path...`);
            args[0].writeUtf8String("/non/existing");
        }
        console.log(`stat: ${args[0].readCString()}`);
    }
})
```

现在是时候再次运行脚本，以查看控制台日志中的更改。

*access: /system/nonexisting*
*fopen:*
*Manipulating selinux path...*
*Error: access violation accessing 0x6d5d9568f8*
    *at <anonymous> (frida/runtime/core.js:147)*
    *at onEnter (/home/kali/Documents/trainings_2023/root_detection_bypass/root_bypass.js:58)*
*detectRoot ret value is 628*

### FRIDA Memory.protect() 修改内存空间权限

在输出中报告了一个错误：“访问0x6d5d9568f8时发生违规访问。”这个错误日志表明，Frida可能试图篡改或覆盖这个特定内存位置上的值。幸运的是，有一个可用的解决方案。我们可以利用另一个Frida API， `Memory.protect()` ，来修改相关内存空间的权限。

```js
Memory.protect(args[0],Process.pointerSize, 'rwx');
```

让我们现在运行脚本，看看是否修复了问题。

*Manipulating su path...*
*access: /system/nonexisting*
*access: ing*
*Manipulating su path...*
*access: /system/nonexisting*
*access: onexisting*
*Manipulating su path...*
*access: /system/nonexisting*
*fopen:*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*fopen: /proc/self/attr/prev*
*detectRoot ret value is 560*

它起作用了！太棒了。在控制台日志中，我们可以观察到 `stat` 输入路径已被修改，如果我们看一下应用程序，它仍然显示为检测到根目录。所以很明显，仅仅绕过这个是不够的，我们需要更详细地分析它，以识别应用程序中存在的其他检查。

## Root Detection 检测 3

在这些状态调用之后的控制台日志中，这次我们又调用了另一个导入的函数，即 `fopen()` ，它试图访问 `/proc/self/attr/prev` 。让我们从adb shell中访问这个文件，看看这个文件中有什么。

```bash
adb shell
cat /proc/self/attr/prev
u:r:zygote:s0
```

让我们在一个非root设备上检查这个相同的文件，并查看其内容。

```bash
cat /proc/self/attr/prev
u:r:untrusted_app:s0:c7,c257,c512,c768
```

好的，这意味着在我们使用已root的设备和未root的设备时，会有一些变化。经过一些研究发现，当我们运行Magisk Zygisk模块时，这个文件将包含 `zygote` 。所以为了绕过这个检查，我们可以禁用Zygisk模块，或者尝试使用Frida绕过它。

### 查看Ghidra中的字符串 zygote

让我们先采用后一种方法。根据我们对该文件的理解，我们可以假设该应用程序在某个时间点上必须进行比较，以检查该文件是否包含“zygote”。让我们再次查看Ghidra中的字符串部分。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_zygote_string.png)

在列出的字符串中，我们确实有 `zygote` 存在。接下来我们需要找出负责进行字符串比较的函数。让我们回到Ghidra中导入的函数。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_imported_strstr.png)

在这里，我们有可用的 `strstr()` 函数，可以用于字符串比较。让我们将我们的钩子附加到这个函数上，并拦截它的参数。

```js
Interceptor.attach(Module.findExportByName("libc.so", "strstr"), {
    onEnter: function (args) {
        console.log(`strstr: haystack -> ${args[0].readCString()} & needle -> ${args[0].readCString()}`);
    }
})
```

**Examine the output: 检查输出结果**

*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*fopen: /proc/self/attr/prev*
*strstr: haystack -> u:r:zygote:s0 & needle -> zygote*
*detectRoot ret value is 571*

**再次运行脚本，看看这次我们得到了什么：**

*Manipulating su path...*
*access: /system/nonexisting*
*access: ing*
*Manipulating su path...*
*access: /system/nonexisting*
*access: onexisting*
*Manipulating su path...*
*access: /system/nonexisting*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*Manipulating selinux path...*
*stat: /non/existing*
*fopen: /proc/self/attr/prev*
*strstr: haystack -> u:r:zygote:s0 & needle -> blabla*
*fopen: /proc/self/mountinfo*
*detectRoot ret value is 532*

我们现在可以观察到，这个第二个论点现在被blabla修改了，但是应用程序仍然不相信，并且仍然检测到有root的设备。

在 `strstr()` 函数调用之后，还有另一个 `fopen()` 调用，然后尝试访问 `mountinfo` 。

## Root Detection 检测 4

当我们拥有一个已经root的设备时，会挂载许多额外的路径，比如Magisk的路径。许多本应只读的路径，比如/system，也变得可写。

因此，该应用程序可能正在尝试寻找这些变化。

由于应用程序可能正在寻找我们在 `strstr()` 函数中识别出的更改，所以它有可能在 `mountinfo `文件中使用相同的函数来查找这些字符串。另一个选择是将 `fopen()` 指向一个不存在的路径，但让我们先尝试第一种方法。由于我们已经将钩子附加到strstr()函数上，所以我们只需要观察输出即可。

```js
fopen: /proc/self/mountinfo
    strstr: haystack -> 20533 20532 253:5 / / ro,relatime master:1 - ext4 /dev/block/dm-5 ro,seclabel
     & needle -> magisk
    strstr: haystack -> 20534 20533 0:18 / /dev rw,nosuid,relatime master:2 - tmpfs tmpfs rw,seclabel,size=2852132k,nr_inodes=713033,mode=755
     & needle -> magisk
    strstr: haystack -> 20535 20534 0:20 / /dev/pts rw,relatime master:3 - devpts devpts rw,seclabel,mode=600,ptmxmode=000
     & needle -> magisk
    strstr: haystack -> 20536 20534 0:19 / /dev/ltgnxkz rw,relatime master:4 - tmpfs magisk rw,seclabel,size=2852132k,nr_inodes=713033,mode=755
     & needle -> magisk
    detectRoot ret value is 655
```

正如我们所怀疑的那样 - 应用程序正在访问mountinfo文件的内容，以搜索这个特定的字符串 `magisk` 。在最后一次迭代中，我们在mountinfo中有这个 `magisk` 。让我们使用之前使用的相同方法来绕过这个问题，修改这个函数的第二个参数。

```js
if(needle.indexOf("magisk") >= 0){
    args[1].writeUtf8String("blabla");
}
```

重新运行脚本后，我们注意到应用现在显示为“可疑”而不是“已root”。这表明我们在某种程度上成功绕过了root检测。

## Root Detection 检测 5

仍然有一些缺失的检查导致应用程序认为设备环境不干净。重新检查输出控制台后，我们没有发现其他值得注意的函数调用。

**那么，还可能是什么呢？**

让我们再次启动Ghidra，并尝试找到其他容易解决的问题。

在随机分析子程序时，我们可以看到一个有趣的指令： `SVC 0x0`.

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_svc_instructions.png)

这些指令在二进制文件中分散，如上面的截图所示。

**这些SVC指令是什么？**

https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#arm64-64_bit
这些指令用于使用系统调用号调用函数。由于我们正在处理Arm64二进制文件，让我们打开该体系结构的系统调用映射表。您可以在此处找到完整的系统调用列表：https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#arm64-64_bit

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_syscall_table.png)

在这个第一列中，我们有函数的名称，然后在第三列中，我们有系统调用号码，并且根据这里的规定，它被存储在 `X8` 寄存器中，然后函数的参数将从 `X0` 到 `X5` 的寄存器中存储。让我们在刚刚看到的反汇编中进行分析。

0x38 存储在 `w8` 寄存器中，我们可以将此系统调用号与该表匹配，以确定调用的是哪个函数。在表中，与此系统调用号对应的函数是 `openat()` 。这个函数类似于 `open()` 函数，它将打开内存中的文件以进行读写操作。这是一个很好的候选项，开发人员使用这种技术来隐藏函数名称，以避免静态分析。

### FRIDA hook svc指令

将Frida拦截器附加到所有这些SVC指令上，我们首先需要找出这些指令存在的偏移量。我们可以借助Ghidra搜索指令模式功能轻松完成这一步骤。

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_ghidra_search.png)

我们找到了所有的SVC指令，如下所示：

让我们现在拦截这些监督员的呼叫，看看 `openat` 函数调用的是哪个文件。

```js
function hookSVC(base_addr){
    Interceptor.attach(base_addr.add(0x00001998), function(){
        var path = this.context.x1.readCString();
        console.log(`svc: ${path}`);
    })

    Interceptor.attach(base_addr.add(0x000019bc), function(){
        var path = this.context.x1.readCString();
        console.log(`svc: ${path}`);
    })

    Interceptor.attach(base_addr.add(0x000019dc), function(){
        var path = this.context.x1.readCString();
        console.log(`svc: ${path}`);
    })

    Interceptor.attach(base_addr.add(0x00001a00), function(){
        var path = this.context.x1.readCString();
        console.log(`svc: ${path}`);
    })

    Interceptor.attach(base_addr.add(0x00001a20), function(){
        var path = this.context.x1.readCString();
        console.log(`svc: ${path}`);
    })
}
```

让我们运行脚本，看看它是否显示出任何有趣的内容。

*svc 56: /system/xbin/su*
*svc 56: /system/bin/su*
*svc 56: /sbin/su*
*svc 56: /system/bin/.ext/su*
*svc 56: /system/sd/xbin/su*
*detectRoot ret value is 566*

从输出中我们可以清楚地观察到这些SVC指令试图访问 `su` 二进制路径。太好了！现在我们可以通过在 `X1` 寄存器中传递一个不存在的路径来轻松绕过这个问题。

```js
Interceptor.attach(base_addr.add(0x000025a0), {
    onEnter: function (args) {
        var path = Memory.readCString(this.context.x1);
        this.context.x1.writeUtf8String("/non/exist");
        console.log(`svc ${this.context.x8.toInt32()}: ${path}`);
    }
})
```

![img](../Root%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87%E6%8A%80%E6%9C%AF.assets/8ksec_root_detector_clean_app.png)

好的，该应用程序不再显示任何投诉，并且现在显示设备环境干净。我们已成功绕过所有检查！

## 结论

在这篇博客文章中，我们了解了Android应用程序中使用的各种检测Root的技术。其中一种技术是检查 `su` 二进制路径的存在。另一种技术是检查selinux策略。我们发现即使绕过了这些检查，该应用程序仍然能够检测到Root。

进一步分析发现了一些基于 `zygote` 和 `mountinfo` 的新检测，这些检测是通过字符串比较进行的。最后，我们发现了一个非常有趣的检测，它是基于 `su` 二进制检测的，但是通过监督指令（ `SVC` ）进行了隐蔽操作。SVC指令用于使用系统调用号调用函数。我们看到了如何识别这些系统调用，并找出与系统调用号对应的函数名称。

最终，我们能够使用Frida作为我们的hooking框架绕过所有这些检查！

> 检测点 函数access 参数路径包含su
> 检测点 函数stat参数路径包含selinux 
> 检测点 函数fopen路径 /proc/self/attr/prev  是否包含zygote
> 检测点 函数strstr参数2 包含zygote
> 检测点 函数fopen路径 /proc/self/mountinfo是否包含magisk 
>
> ```javascript
> svc 56: /system/xbin/su
> svc 56: /system/bin/su
> svc 56: /sbin/su
> svc 56: /system/bin/.ext/su
> svc 56: /system/sd/xbin/su
> ```
