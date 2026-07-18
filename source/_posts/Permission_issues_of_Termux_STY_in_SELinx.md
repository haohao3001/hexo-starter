---
title: 'Termux的伪终端在SELinux下的权限问题'
---
---
现在手机里的狗屎应用也是越来越多了，一堆系统软件想删还得用`shell(uid=2000)`删，本来想用termux(这里的su是切换到shell(uid=2000)，不是root(uid=0))执行`pm`给删的，结果遇到了神秘的权限问题`cmd: Failure calling service package: Failed transaction (2147483646)`，最后还是把AppProfile设为默认(root)才解决的

## 1.寻找是哪出了问题
虽然是个小问题，但既然有问题，那无疑是要解决的  
处理权限的就只有DAC和MAC(SELinux)，那首要的就是判断是不是DAC引起的，对照着`adb shell`的gid，在KernelSU里设置对应的AppProfile，最后su提权测试，不出所料，还是报错，看来是SELinux干的了，那说明肯定是某个东西的context不对导致的，用dmesg查看日志，有`system_server` `untrusted_app_all_devpts`这个context，那明了了，SELinux没有配置这两个context间的权限，因此按照默认拒绝的原则，直接返回Permission Denid了

## 2.解决问题
### 1.1 方法一
既然是SELinux，那直接把SELinux设为Permissive即可(大雾)，这显然是不可行的，没了SELinux那软件不随便就能提权了

### 1.2 方法二
把AppProfile改回默认，在以root执行pm时不会出现这个问题，原理我也搞不清楚，但这对有洁癖的人无疑是不友好的

### 1.3 方法三
既然是system_server(u:r:system_server)与TermuxPTY(u:r:untrusted_app_all_devpts)之间的通信问题，那直接绕过伪终端，加个中间件即可，这时就要拿出管道了，system_server与伪终端的问题关管道什么事，用管道桥接一下即可解决  
```bash
#!/data/data/com.termux/files/usr/bin/bash

# 获取当前脚本的执行名称 (例如: pm, am, wm...)
BIN_NAME=$(basename "$0")

su -c "/system/bin/$BIN_NAME $@ < /dev/null 2>&1 | cat"
```
把通用脚本放在/data/data/com.termux/files/usr/local/bin下并命名为pm，然后在`.bashrc`里加上`PATH="/data/data/com.termux/files/usr/local/bin:$PATH"`即可解决，之后在termux里直接执行pm即可