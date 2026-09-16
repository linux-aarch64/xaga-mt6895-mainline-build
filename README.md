# xaga-mt6895-mainline-build

基于 GitHub Actions 的 **Redmi Note 11T Pro (xaga / MT6895 / 天玑8100)** 主线内核自动化构建项目。

内核源码: [MT6895-Mainline/linux (7.2-mt6895-xiaomi-xaga)](https://github.com/MT6895-Mainline/linux/tree/7.2-mt6895-xiaomi-xaga)
Initramfs: [MT6895-Mainline/initramfs](https://github.com/MT6895-Mainline/initramfs)

## 项目结构

```
xaga-mt6895-mainline-build/
├── .github/
│   └── workflows/
│       ├── build.yml    # 主编译工作流 (内核编译 + RootFS构建 + 完成通知)
│       └── clean.yml    # 缓存清理工作流
├── bot.py               # 自动化助手 (钉钉/Server酱通知 + 上游更新检测 + 自动触发构建)
└── README.md
```

## 功能特性

| 功能 | 说明 |
|------|------|
| Clang 全链路编译 | LLVM=1, 作者强制规范, 禁用 GCC |
| Image+DTB 拼接 | 绕过原厂 LK dtbo 限制 |
| 项目专用 initramfs | 使用 MT6895-Mainline/initramfs|
| 多发行版 RootFS | postmarketOS / Arch Linux ARM / Ubuntu 三选一 |
| 钉钉/QQ 通知 | 构建完成自动推送 (钉钉 Webhook / Server酱) |
| 上游更新监控 | bot.py 自动检测上游内核 commit, 可自动触发构建 |

## 使用方法

### 1. 触发构建

仓库页面 → **Actions** → 选择 **Build** → 点击 **Run workflow**, 填写参数:

| 参数 | 选项 | 说明 |
|------|------|------|
| 构建任务类型 | 仅编译内核 / 仅构建RootFS / 全部 | 选择要执行的任务 |
| RootFS发行版 | postmarketOS / Arch Linux ARM / Ubuntu | 仅 RootFS 任务生效 |
| postmarketOS桌面 | phosh / none / plasma-mobile | 仅 pmOS 生效 |
| Clang版本 | 默认 18 | LLVM 编译版本 |

### 2. 下载产物

构建成功后, 在任务页面底部 **Artifacts** 下载:
- `kernel-output`: `boot.img` + `Image_with_dtb` + `initramfs.cpio.lz4`
- `rootfs-output`: `rootfs.img` (ext4 根文件系统镜像)

### 3. 刷写

```bash
# 刷内核 (boot 分区)
fastboot flash boot boot.img

# 刷 RootFS (userdata 分区, 注意会清空安卓数据!)
fastboot flash userdata rootfs.img

fastboot reboot
```

> userdata 物理分区已确认为 `/dev/sdc86` (主线内核命名), cmdline 已内置。

### 4. 清理缓存

当编译异常或缓存冲突时, Actions → **Clean Cache** → Run workflow, 一键清空全部缓存

## 分区确认

本设备 userdata 物理分区编号已通过 `/proc/partitions` 确认:

| 环境 | 设备名 |
|------|--------|
| Android (原厂内核) | `/dev/sdc86` |
| 主线内核 (MT6895-mainline) | `/dev/sdc86` |

> 分区号数字不变, userdata对应分区编号为/dev/sdc86。cmdline 中使用主线内核命名 `/dev/sdc86`。

## 注意事项

1. **必须解锁 Bootloader** 才能刷入自定义 boot.img
2. `fastboot flash userdata` 会彻底清除安卓用户数据, 操作前务必备份
3. 官方 initramfs 当前原生仅支持物理分区挂载,
4. 首次进入 postmarketOS 后执行 `sudo apk add linux-firmware-mediatek` 补全固件
5. 救砖: fastboot 刷回原厂 boot.img; 若 userdata 已覆盖需 MiFlash 线刷整机

## 技术栈

- 编译: Clang-18 / LLVM / LLD (全 LLVM 工具链)
- 打包: osm0sis mkbootimg (兼容 MTK 原厂 LK)
- RootFS: postmarketOS edge / Arch Linux ARM / Ubuntu 24.04 / Debian 13
- 通知: 钉钉 Webhook (加签) / Server酱
- 缓存: actions/cache (内核 obj + RootFS 镜像)
