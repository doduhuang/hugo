---
title: adb命令大全
categories: # 分类
	- Android # 只能由一个
tags: # 标签
	- adb # 能有多个
	 # - hook一个标签一行
---

###### 常规ADB命令

```sh
//关闭adb
adb kill-server

//开启adb
adb start-server

//查看当前连接设备
adb devices

//如果发现多个设备 例：adb -s devicel install xxx.apk
adb -s 设备号 其他指令

//往手机SDCard传递文件
adb push 文件名 手机端SDCard路径

//从手机端下载文件
adb pull /sdcard/xxx.txt
```

###### 操作APP

```sh
//查看手机端安装的所有app包名
adb shell pm list packages

//查看顶部Activity
adb shell dumpsys activity | findstr "mFocusedActivity"

//安装apk文件 //使用覆盖安装：adb install -r xxx.apk
adb install xxx.apk

//卸载App:   //保留数据: adb uninstall -k com.zhy.app
adb uninstall com.zhy.app

//启动Activity 例:adb shell am start com.zhy.aaa/com.zhy.aaa.MainActivity
adb shell am start 包名/完整Activity路径

//启动一个隐式的Intent:
adb shell am start -a "android.intent.action,VIEW" -d "https://www.google.com"

//发送广播 需要携带参数（携带一个Intent,key为name）:adb shell am broadcast -a "broadcastactionfilter" -e name zhy
adb shell am broadcast -a "broadcastactionfilter"

//启动服务
adb shell am startservice "com.zhy.aaa/com.zhy.aaa.MyService"

//屏幕截图
adb shell screencap /sdcard/screen.png

//录制视频
adb shell screenrecord /sdcard/demo.mp4

//清除APP数据
adb shell pm clear com.example.packagename


```

###### 使用adb为应用程序授予权限adb install 指令如下，

adb install -r 替换已存在的应用程序，也就是说强制安装
adb install -l 锁定该应用程序
adb install -t 允许测试包
adb install -s 把应用程序安装到sd卡上
adb install -d 允许进行将见状，也就是安装的比手机上带的版本低
adb install -g 为应用程序授予所有运行时的权限

下面讨论为应用程序授予权限的使用场景，

1. 对应用程序授予所有的运行时的权限
   $ adb install -g xxx.apk
2. 对于某些权限，如“MANAGE_EXTERNAL_STORAGE”无法使用“-g”授予的，可以使用如下命令
   $ adb shell appops set --uid com.company.name MANAGE_EXTERNAL_STORAGE allow
3. 还可以使用如下命令单独授予应用程序某一个权限，但是“MANAGE_EXTERNAL_STORAGE”无法授予权限
   $ adb shell pm grant com.comany.name android.permission.CAMERA



###### 安卓日志adblogcat

adb logcat [选项] [过滤项], 其中 选项 和 过滤项 在 中括号 [] 中, 说明这是可选的;

选项解析:

- "-s"选项 : 只显示指定标签的日志; ------>adb logcat -s SWVDEC 显示SWVDEC标签的日志

- "-v"选项 : 设置日志的输出格式;----->adb logcat -v threadtime 查看日志输出时间和线程信息

- "-c"选项 : 清空所有的日志缓存信息;---->adb logcat -c

- "-d"选项 : 将缓存的日志输出到屏幕上, 并且不会阻塞;------->adb logcat -d 

- "-t"选项 : 输出最近的几行日志, 输出完退出, 不阻塞;------>adb logcat -t 5 输出日志缓冲区的最近5行

- "-g"选项 : 查看日志缓冲区信息; ------>adb logcat -g

- "-B"选项 : 以二进制形式输出日志; ----> adb logcat -B

log输出到计算机文件 adb logcat > /home/cherish/log.txt ( 相对路径.绝对路径都可以.)

过滤指定标签"android"或者"system"标签的log,并输出到文件:
adb logcat | grep -E “android|system” > /home/cherish/log.txt

将日志保存到文件 , 但无法输出到屏幕 , 针对这个问题可以采用 tee命令 .
adb logcat | grep -E “android|system” | tee /home/cherish/log.txt
