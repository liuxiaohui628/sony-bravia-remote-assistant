# Aerial 桌面配置

本项目把索尼原生桌面换成了 **Aerial**（`app.orchard.tv`），并做了一轮应用精简。本文记录具体做法，克隆项目后照做即可得到同样的桌面。

---

## 一、为什么换 Aerial

| 对比项 | 原生 Google TV 桌面 | Aerial |
|---|---|---|
| 首屏信息 | 大量推荐位（大多无法关闭） | 纯应用网格，干干净净 |
| 自定义 | 只能调顺序 | 可隐藏任意应用、自定义壁纸、调整行列数 |
| 性能 | 推荐位会持续拉取网络内容 | 静态渲染，响应快 |
| 体积 | 系统内置 | 约 4 MB |

安装：

```bash
adb install -r Aerial.apk
adb shell cmd package set-home-activity app.orchard.tv/.MainActivity
```

---

## 二、应用精简清单

桌面只保留 **10 个**真正会用的应用，其余全部隐藏。

### 保留的应用

| # | 应用 | 包名 | 用途 |
|---|---|---|---|
| 1 | 影视仓 | `com.huawei.himovceie` | 主力影视（电视键默认目标） |
| 2 | 云视听小电视 | `com.xiaodianshi.tv.yst` | 直播/点播（视频键默认目标） |
| 3 | WebView 电视 | `com.vasthread.webviewtv.v2` | 网页浏览 |
| 4 | 夸克网盘 | `com.quark.yun.tv` | 网盘播片 |
| 5 | AirPlay Server | `io.github.jqssun.airplay` | 投屏接收（见 [AIRPLAY.md](AIRPLAY.md)） |
| 6 | Clash | `com.github.metacubex.clash` | 网络代理（看境外内容） |
| 7 | ES文件浏览器 | `com.estrongs.android.pop` | 本地文件管理 |
| 8 | Alook | `alook.browser` | 轻量浏览器（备用） |
| 9 | Probe | `com.wb.keyprobe` | 本项目（语音助手/按键/扫码） |
| 10 | Settings | `com.android.tv.settings` | 系统设置 |

### 隐藏的应用

原生系统预装但日常不用的，全部从桌面隐藏（不卸载，避免系统异常）：

| 应用 | 包名 | 隐藏原因 |
|---|---|---|
| Google TV 推荐主页 | `com.google.android.apps.tv.launcherx` | 被 Aerial 替代 |
| Play 影视 | `com.google.android.videos` | 国内不可用 |
| Play 商店 | `com.android.vending` | 国内需代理，用第三方市场替代 |
| YouTube | `com.google.android.youtube.tv` | 国内不可用 |
| YouTube Music | `com.google.android.youtube.tvmusic` | 同上 |
| Google 相册 | `com.google.android.apps.photos.tv` | 不用 |
| 索尼精选 | `com.sony.dtv.sonyselect` | 内容少 |
| 索尼推荐 | `com.sony.dtv.recommendation` | 广告性质 |
| 电视直播 | `com.sony.dtv.tvx` | 不看有线电视 |
| 音乐 | `com.sony.dtv.music` | 用手机播 |
| 相册 | `com.sony.dtv.album` | 不用 |
| 帮助指南 | `com.sony.dtv.helpguide` | 不用 |

---

## 三、如何隐藏应用（Aerial 内置功能）

```
Aerial 设置（长按主页 或 桌面底部齿轮）
→ Apps
→ 找到目标应用，取消勾选「Visible」
```

批量隐藏可以用 adb（Aerial 的配置存在 SharedPreferences）：

```bash
# 查看当前 Aerial 配置
adb shell run-as app.orchard.tv cat shared_prefs/*.xml
```

> 注意：Aerial 的隐藏只是「不在桌面显示」，应用本身仍可被语音或按键启动。这对我们很重要——影视仓虽然是默认目标，但桌面上保留它方便手动打开。

---

## 四、一键复刻（脚本）

把下面存为 `setup-desktop.sh`，在电视上执行即可完成桌面配置：

```bash
#!/system/bin/sh
# 1. 设为默认桌面
cmd package set-home-activity app.orchard.tv/.MainActivity

# 2. 隐藏系统预装（Aerial 不显示）
#    注意：这里只是移出桌面，不卸载
for pkg in \
  com.google.android.apps.tv.launcherx \
  com.google.android.videos \
  com.android.vending \
  com.google.android.youtube.tv \
  com.google.android.youtube.tvmusic \
  com.google.android.apps.photos.tv \
  com.sony.dtv.sonyselect \
  com.sony.dtv.recommendation \
  com.sony.dtv.album \
  com.sony.dtv.helpguide ; do
    cmd package set-enabled-setting "$pkg" default 2>/dev/null
done

echo "桌面配置完成，重启 Aerial 生效"
am force-stop app.orchard.tv
am start -n app.orchard.tv/.MainActivity
```

---

## 五、壁纸与布局

| 项 | 推荐值 |
|---|---|
| 壁纸 | 深色抽象图（避免纯黑，Aerial 图标是浅色） |
| 列数 | 6 列（65 英寸最舒适） |
| 行数 | 3 行 |
| 图标大小 | 中等 |
| 显示应用名 | 开 |

---

## 六、和按键改造的协同

桌面精简后，配合按键改造形成完整闭环：

```
电视键 ──▶ 影视仓（跳过桌面找应用的步骤）
视频键 ──▶ 云视听
主页键 ──▶ 回 Aerial 桌面（10 个应用一目了然，不用翻页）
屏显键 ──▶ 扫码输入（不用遥控器打字）
语音键 ──▶ 直接说需求，连桌面都不用看
```

**实际体验**：绝大多数场景下，从「想看」到「看上」只需要按一个键 + 说一句话，桌面基本只在切换不常用应用时才会出现。
