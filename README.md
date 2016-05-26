# android_source

copy from luo
h1. Android gdb debugging

Android provides great development tools and makes great use of the available Java tools 
like eclipse debugging and junit test.. It also provide stack-traces in it's log and tool to
upload crashed to a central server. These tools will work for you most of the time but not
always. To be able to debug SEGFAULTS and such it can be very handy to have a proper gdb 
setup in place. Android provide good support and as usual poor documentation on how to do this.


h2. The setup


```
 target file system                               host file system
    target process                         host files containing debugging 
       |                                symbols out/device/symbols out/device
       |                              (e.g. memory address to code line mapping)
debugging syscall (ptrace)                              |
       |                                                |
       |----Process to debug                            |
       |                                                |
    gdb server                        gdb with support for the target architecture.
       |                                                |
      Socket                                          Socket
       |                                                |
      adbd multiplex----------------------------adbd multiplex
```

h2. Sample debugging the system_server(2.3)


```
kejo@kejo-fakedistro:~$ adb forward tcp:5039 tcp:5039
kejo@kejo-fakedistro:~$ adb shell ps  | grep system_ | awk '{print $2}'
63
kejo@kejo-fakedistro:~$ adb shell gdbserver :5039 --attach 63
Attached; pid = 63
Listening on port 5039
Remote debugging from host 127.0.0.1
gdb: Unable to get location for thread creation breakpoint: requested event is not supported
```


```
kejo@kejo-fakedistro:~/projects/android-gingerbread$ gdbclient 

If you haven't done so already, do this first on the device:
    gdbserver :5039 /system/bin/app_process
 or
    gdbserver :5039 --attach 

GNU gdb (GDB) 7.1-android-gg2
Copyright (C) 2010 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=x86_64-linux-gnu --target=arm-elf-linux".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /home/kejo/projects/android-gingerbread/out/target/product/generic/symbols/system/bin/app_process...done.
BFD: /home/kejo/projects/android-gingerbread/out/target/product/generic/symbols/system/bin/linker: warning: sh_link not set for section `.ARM.exidx'
Error while mapping shared library sections:
gralloc.default.so: No such file or directory.
Error while mapping shared library sections:
gps.goldfish.so: No such file or directory.
__ioctl () at bionic/libc/arch-arm/syscalls/__ioctl.S:15
15      ldmfd   sp!, {r4, r7}
(gdb) bt
#0  __ioctl () at bionic/libc/arch-arm/syscalls/__ioctl.S:15
#1  0xafd25ca8 in ioctl (fd=-512, request=1) at bionic/libc/bionic/ioctl.c:41
#2  0x80419fd6 in android::IPCThreadState::talkWithDriver (this=0x8d900, 
    doReceive=<value optimized out>)
    at frameworks/base/libs/binder/IPCThreadState.cpp:791
...
#9  0xaca16f98 in dalvik_mterp ()
    at dalvik/vm/mterp/out/InterpAsm-armv5te.S:10541
#10 0xaca1c0a0 in dvmMterpStd (self=<value optimized out>, glue=0xbeeaf500)
---Type <return> to continue, or q <return> to quit---q
 at dQuit
(gdb) detach
Ending remote debugging.
(gdb) quit
```

h2. Sample debugging the zygote startup

To debug zygote it's the best to modify init.rc to start zygote directly using gdbclient

```
    service zygote /system/bin/gdbserver :5039 /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```


h2. Attaching to processes that crashes (Just in time debugging)

You can set system settings in Android to it will suspend processes that crash so you
can attach to them. adb shell setprop debug.db.uid 32767 will cause the crash catcher (debuggerd) to freeze the process

Valid values for uid if set (> 0) wait for the user action for the crashing process
if the pid of the process is bigger then the  uid siblings are dumped

http://www.kandroid.org/online-pdk/guide/debugging_native.html has more nice info/tips about 
using set scheduler-locking on,handle SIGUSR1 noprint

h2. Debugging other processses

If you wan to debug other processes it is best to just run gdbclient
and set the file begin debugged using the file command or calling gdbclient with an argument

adb forward tcp:5039 tcp:5039
adb shell gdbserver :5039 --attach \`pidof rild\` (Android 2.2) in Android 2.3 pidof was removed....
gdbclient rild

h2. Parsing the log backtraces using addr2line

If you can not directly debug the progress you can do the following

1) Using addr2line to get a backtrace

If you get a backtrace in a log can convert the addresses back to source using addr2line

```
Apr 21 14:38:27 I/DEBUG   (  946):          #03  pc 0005946e  /system/lib/libdvm.so
```

```
. build/envsetup.sh
lunch mytarget-eng << change this
cd $OUT
arm-eabi-addr2line -fC -e symbols/system/lib/libdvm.so  0005946e
dvmInvokeMethod
/home/kejo/projects/android-gingerbread/dalvik/vm/interp/Stack.c:726
```

h2. System properties you can use to debug

setprop debug.db.uid 32767
setprop dalvik.vm.checkjni true #enables jni checks (security)
setprop dalvik.vm.jniopts forcecopy

#Enable malloc debugging
setprop libc.debug.malloc 10


h2. Tools you can use

* dumpsys is a great tool to get a view of the system state it dumps every system service. It dumps all threads in the system.
