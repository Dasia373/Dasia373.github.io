---
date: 2026-03-13
draft: false
title: "韦东山IMX6ULL-PRO入门学习日志（二）替换与编译内核"
tags: ["Linux", "WSL2", "ADB", "IMX6ULL-PRO"]
categories: ["Linux"]
---

## 1. 清理源码目录

```
make mrproper
```

------

## 2. 加载默认配置

```
make 100ask_imx6ull_defconfig
```

------

## 3. 编译内核镜像

```
make zImage -j4
```

这里 `-j4` 表示使用 4 个线程并行编译。

------

## 4. 编译设备树

```
make dtbs
```

编译完成后，设备树文件会生成在：

```
arch/arm/boot/dts/
```

这里使用的设备树文件是：

```
arch/arm/boot/dts/100ask_imx6ull-14x14.dtb
```

------

## 5. 将编译结果通过 ADB 拷贝到开发板

编译完成后，可以直接通过 ADB 将内核、设备树和模块下发到开发板。

### 拷贝内核镜像

```
adb push arch/arm/boot/zImage /boot
```

### 拷贝设备树

```
adb push arch/arm/boot/dts/100ask_imx6ull-14x14.dtb /boot
```

### 拷贝模块目录

```
adb push lib/modules /lib
```

------

## 6. 登录开发板并重启

进入开发板 shell：

```
adb shell
```

在开发板中执行：

```
sync
reboot
```

`sync` 用来将缓存中的数据写回存储设备，确保刚刚通过 ADB 下发的文件已经落盘。

------

## 7. 重启后验证内核版本

开发板启动完成后，再次登录 shell：

```
adb shell
```

查看当前内核版本：

```
uname -a
```

如果输出中包含 `4.9.88`，说明新的内核已经启动成功。

------

## 8. 本节完整命令汇总

```
cd /home/book/100ask_imx6ull-sdk/Linux-4.9.88
make mrproper
make 100ask_imx6ull_defconfig
make zImage -j4
make dtbs

adb push arch/arm/boot/zImage /boot
adb push arch/arm/boot/dts/100ask_imx6ull-14x14.dtb /boot
adb push lib/modules /lib

adb shell
sync
reboot

adb shell
uname -a
```