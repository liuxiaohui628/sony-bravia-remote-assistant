# AirPlay 投屏方案

把电视变成 iPhone / Mac 的 AirPlay 接收端。索尼 XR-65X90J 硬件支持 AirPlay 2，但国行固件默认不开放，需要装第三方接收端。

---

## 一、为什么需要额外装

| 事实 | 说明 |
|---|---|
| 硬件支持 | 规格表标注 `APPLE AIRPLAY: Yes`，芯片与射频具备条件 |
| 国行限制 | 国行系统（`BRAVIA_VH2_M_CN`）未内置 AirPlay 接收端 |
| 系统自带 | 「屏幕镜像」功能只支持 Miracast（安卓协议），不认 AirPlay |
| 结论 | 需要第三方 AirPlay Server 弥补 |

---

## 二、采用的方案

**android-airplay-server**（开源项目 `jqssun/android-airplay-server`）

| 项 | 值 |
|---|---|
| 包名 | `io.github.jqssun.airplay` |
| 版本 | v0.0.36 |
| 原理 | 在电视上跑一个 AirPlay 接收服务（RTSP + H.264 解码 + 音频输出） |
| 依赖 | root（su）权限、SELinux 宽松模式 |
| 体积 | 约 3 MB |

### 为什么选它

- 纯本地运行，不走云端中转，延迟低
- 支持镜像 + 音频串流
- 开源可审计，不会偷偷上传画面

### 已知短板

- 4K 素材会降采样到 1080p（电视网络带宽 + 解码能力限制）
- 首次连接有 2-3 秒握手延迟
- iOS 17 之后部分设备需要重新配对

---

## 三、部署步骤

```bash
# 1. 安装 APK
adb install -r android-airplay-server-v0.0.36.apk

# 2. 授予 root 执行权限（电视需已 root）
adb shell su -c "pm grant io.github.jqssun.airplay android.permission.RECORD_AUDIO"

# 3. 关闭 SELinux 强制模式（否则服务无法绑定 7000 端口）
adb shell su -c "setenforce 0"

# 4. 启动服务
adb shell am start -n io.github.jqssun.airplay/.MainActivity
```

### 连接方式

iPhone / iPad：

```
控制中心 → 屏幕镜像 → 选择「BRAVIA 4K VH2」
```

Mac：

```
菜单栏控制中心 → 屏幕镜像 → 选择电视
```

---

## 四、与语音助手联动

语音指令已支持：

| 说 | 效果 |
|---|---|
| 打开隔空播放 | 启动 AirPlay Server |
| 打开屏幕镜像 | 同上传 |
| 投屏 | 同上传 |

实现方式：在 `handleCommand` 里匹配关键词后，用包名直接启动：

```java
launchPkg("io.github.jqssun.airplay");
```

---

## 五、踩坑记录

| 现象 | 原因 | 解决 |
|---|---|---|
| 启动后立即退出 | SELinux 拦截端口绑定 | `setenforce 0` |
| iPhone 搜不到设备 | 电视与手机不在同一网段 | 检查是否连同一个 Wi-Fi（2.4G/5G 同名不同网段也会失败） |
| 连上后黑屏有声音 | 编码格式不兼容 | 在服务设置里切换到 H.264 baseline |
| 画面卡顿 | Wi-Fi 带宽不足 | 优先用 5GHz；或改用有线网口 |
| 重启电视后失效 | `setenforce 0` 不持久 | 需要在开机脚本里重新执行，或改用 permissive 内核参数 |

---

## 六、和「扫码输入」的关系

AirPlay 开启时，电视处于「外部输入」状态，此时：

- 二维码浮窗仍然可以显示（悬浮窗层级高于投屏画面）
- 但手机端触摸板控制的是 AirPlay 画面，不是电视系统 —— **无法操作投屏内容**
- 语音助手也会失效（系统被投屏占用音频通道）

**建议**：投屏时不要开扫码输入，两者场景互斥。
