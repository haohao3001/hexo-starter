---
title: '为REDMI K90 ProMax适配OrangeFox'
---
---
已经做好的设备树[android_device_xiaomi_myron
](https://github.com/haohao3001/android_device_xiaomi_myron)
  
设备: REDMI K90 Pro Max，代号 myron，8Elite5芯片  
基版: [OrangeFox fox_16.0](https://github.com/OrangeFox16/sync) (Android16)

---

## 1.寻找现成的轮子(适配前的准备工作)
拿到设备后当然不可能是从零开始造轮子的(这太累了说是)

### 1.1认识设备的参数
| 项目 | 值 | 影响 |
| :--- | :---: | ---: |
| Soc | 8Elite5 | 影响到解密Data |
| 分区方案 | Virtual A/B + 独立 recovery 分区 | 不需要 `BOARD_USES_RECOVERY_AS_BOOT` |
| 内核 | GKI6.12 | 内核在 vendor_boot，recovery 镜像不含内核 |
| 屏幕 | 1200*2608,120Hz | 需要配置 BoardConfig:`TARGET_SCREEN_WIDTH` `TARGET_SCREEN_HEIGHT`<br>device.mk:`OF_SCREEN_H` `OF_STATUS_H` |
| 振动 | qcom-hv-haptics | 需要特殊处理(后面有讲) |

### 1.2寻找现成的轮子
變換風雲@coolapk 的TWRP: 能直接复用/参考里面的二进制  
hackpupg001-a11y的设备树: 能参考里面的BoardConfig.mk  
OnePlus 15等Android16设备: 可参考里面解密Data的实现  
可参考的文章: [橙狐适配指南：为 Redmi K60 Pro (socrates) 移植OrangeFox Recovery](https://www.cnblogs.com/mmjio/p/20034307)

### 1.3编译环境
```
ArchLinux，任一Linux发行版即可(WSL2也行)
CPU: i5-12490f
RAM: 32G(构建时也会吃内存)
Disk: 120G可用(至少预留90G)
网络: 一个稳定的科学上网工具，需要从googlesource下载大约80G的源码(实际消耗约20G左右的流量)
```

## 2.适配策略
这个在[橙狐适配指南：为 Redmi K60 Pro (socrates) 移植OrangeFox Recovery](https://www.cnblogs.com/mmjio/p/20034307)内有详细说明，因此这里就不水字数了，后面详细讲一下关于解密Data，震动，假回锁处理

## 3.踩坑记录

### 3.1 实现解密Data
高通方案下，需要用到`qseecomd` `hlosminkdaemon`这两个文件极其配套的service配置文件`qseecomd.rc` `hlosminkdaemon.rc`，都可以在vendor里找到的  
对于这两个文件依赖的库，可以通过`readelf -d your/program | grep NEEDED`查看  
高版本Android还需要在BoardConfig.mk中设置(TW_INCLUDE_OMAPI := true)  
(当时找了好久原因说是，最后还是在翻看OnePlus 15的设备树时发现的)

### 3.2 震动
最初的想法是补全震动所需的二进制，跑AIDL vibrator service，结果尝试的半天也没成功说是  
最后才是参考[OrangeFox 适配 Redmi K60 Pro：从源头解决 Recovery 点击卡顿和振动问题](https://www.cnblogs.com/mmjio/p/20034309)才发现还能用force-feedback解决

### 3.3 假回锁处理
编译出的OrangeFox不能直接刷入recovery分区，不然会在重启时卡在fastboot或无限重启  
这里我猜测一下原理应该是gbl root canoe只处理了比对哈希的部分，没有处理用公钥解密尾部的AVB信息这一步，公钥解密失败就直接拒绝引导内核了  
解决方法也很简单，把官方的recovery的AVB尾部移植过去即可，这里可以用[我的脚本](https://github.com/haohao3001/android_device_xiaomi_myron/blob/main/transplanting_vbmeta.py)  

### 3.4 卡在Fastboot了怎么办
这个时候千万不要说去擦除efisp，降级/升级abl，xbl，不然成砖了都没得哭的，还会被群主设为精华(不是)  
把损坏的上层分区(HLOS)(通常是recovery分区里的文件损坏)重刷回官方镜像，然后在官方的fastboot里执行`fastboot set_active a`重置计数器  
最后执行`fastboot reboot`或`fastboot continue`即可  
**千万不要在superfastboot里执行set_active**  
**千万不要在superfastboot里执行set_active**  
**千万不要在superfastboot里执行set_active**  
**不然直接喜提黑砖**

### 3.5 MTP没得用
某些情况下把TWRP的二进制搬到OrangeFox后会导致MTP没得用的问题，这不是BoardConfig.mk也不是二进制文件的问题  
是`init.recovery.usb.rc`和`system/etc/init/hw/init.rc`同时实现了USB控制器   
以及重复导入`init.recovery.usb.rc`(init.rc和init.recovery.qcom.rc都申明了`import /init.recovery.usb.rc`)导致的  
把删除了USB逻辑的`init.rc`放在`system/etc/init/hw/`覆盖掉自动生成的`init.rc`，然后检查一下有没有重复导入`init.recovery.usb.rc`的情况  
最后重新编译即可(可增量编译的)
