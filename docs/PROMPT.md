# 提示词描述（复刻指南）

把下面这段直接发给 AI，可以复现一个类似项目。已按实际有效的表述整理。

---

## 一、完整项目提示词

```
帮我做一个 Android TV 无障碍服务应用，目标是把索尼电视的遥控器改成智能中控。

技术约束：
- 纯 Java，不用 Gradle，用 javac + d8 + jarsigner 手工打包（提供 build.sh）
- minSdk 26，targetSdk 28，类型为 AccessibilityService
- 所有 UI 用代码绘制，不用 XML 布局

需要实现四块功能：

【一】语音助手
1. 拦截遥控器语音键（KEYCODE_SEARCH=84），按下开始 SpeechRecognizer 录音，松手结束
2. 说话时显示 Siri 风格 HUD：底部流体彩色光晕 + 白色文字胶囊 + 蓝色麦克风胶囊（左右各 7 根声波柱）
3. 识别结果优先填入当前焦点输入框（ACTION_SET_TEXT），没有输入框则当指令执行
4. 指令支持：打开应用、音量调节、主页/返回/最近任务、影视搜索、卸载应用

【二】自定义按键
1. 电视键(170)、视频键(203)：短按打开绑定的应用，长按 2 秒弹出应用选择器
2. 长按过程显示圆形倒计时（圆圈 + 数字 + "N 秒后设置按键" + "松开取消"）
3. 倒计时延迟 200ms 才出现，避免快速点按闪一下
4. 已在绑定应用内时，短按不响应
5. 长按期间忽略其他按键
6. 主页键长按 2 秒清理后台并洒下粉色彩带

【三】扫码输入
1. 按屏显键(INFO=165)在左下角弹出二维码浮窗
2. 内置 ServerSocket HTTP 服务（8080 端口），手机扫码打开网页
3. 网页输入实时同步到电视输入框（停顿 280ms 自动提交）
4. 网页包含触摸板：单指移动光标、轻点单击、长按、双指滚动、底部三大导航键

【四】稳定性
1. 语音响应要快：松手时若有文本立即执行，不等最终结果
2. 松手后 1.2 秒必收起（防卡住）
3. 识别器忙时自动重试，引擎无响应时销毁重建
```

---

## 二、关键技术提示词（分模块）

### 拦截遥控器按键

```
AccessibilityService 拦截电视遥控器按键，需要：
1. res/xml 配置 android:canRequestFilterKeyEvents="true"
2. android:accessibilityFlags 加 flagRequestFilterKeyEvents
3. onKeyEvent 里 return true 才算拦截，return false 事件继续传递

注意：KEYCODE_TV(170) 会被系统 TV Input 优先消费，
无障碍服务能收到事件但要用 return true 才能阻止系统默认行为。
```

### 调用索尼语音识别

```
电视没有内置麦克风，遥控器语音走索尼私有通道。
用 SpeechRecognizer 而不是 AudioRecord——前者能拿到遥控器麦克风的数据。

Intent 参数：
- ACTION_RECOGNIZE_SPEECH
- EXTRA_LANGUAGE = "zh-CN"
- EXTRA_PARTIAL_RESULTS = true
- EXTRA_SPEECH_INPUT_COMPLETE_SILENCE_LENGTH_MILLIS = 1800
```

### 模拟鼠标点击

```
必须同时满足三个条件，否则 dispatchGesture 静默失败：
1. accessibility_service_config.xml 加 android:canPerformGestures="true"
2. 有 SYSTEM_ALERT_WINDOW 权限（显示光标）
3. dispatchGesture 的坐标是屏幕绝对坐标，不是 View 坐标

验证方法：dumpsys accessibility | grep capabilities
出现 32（CAPABILITY_CAN_PERFORM_GESTURES）才算生效。
```

### 内置 HTTP 服务

```
不要用 NanoHTTPD 等库，手写 ServerSocket 更可控：

1. ServerSocket(8080) 在独立线程 accept
2. 逐字节读请求头，读到 \r\n\r\n 为止（BufferedReader 读 body 不可靠）
3. 按 Content-Length 读 body 字节
4. 返回时手动拼 HTTP/1.1 响应头

踩坑：用 BufferedReader 读 body 会吞掉字节，
导致 POST 提交内容为空——必须用 InputStream 按字节读。
```

### 网页实时同步

```
输入框监听 input 事件，做 280ms 防抖后 POST 到电视：

let timer = null;
t.addEventListener('input', () => {
  clearTimeout(timer);
  timer = setTimeout(push, 280);
});

不要做「发送按钮」，实时同步体验更好。
```

---

## 三、踩坑提示词（环境问题）

### 应用网络被切断

```
现象：HttpURLConnection / ServerSocket 全部报 EPERM (Operation not permitted)
诊断：adb shell am get-standby-bucket <包名>
      返回 5 = RESTRICTED（受限级，socket 被系统切断）

解决：
- am set-standby-bucket active 无效（被硬锁定）
- deviceidle whitelist 无效
- 唯一有效：卸载重装，bucket 会重置为 50 (EXEMPTED)
```

### 重装后权限丢失

```
卸载重装会清空所有运行时权限，需要重新授权：
adb shell pm grant <包名> android.permission.RECORD_AUDIO
adb shell appops set <包名> SYSTEM_ALERT_WINDOW allow
adb shell settings put secure enabled_accessibility_services <包名>/<服务类>
adb shell settings put secure accessibility_enabled 1
```

### 影视仓自定义键盘

```
影视仓的搜索键盘不是系统输入法，TYPE_INPUT_METHOD 窗口检测不到。
改用界面内容判断：遍历节点找 text 或 hintText 含「请输入」的即可。
```

---

## 四、可复用的分析思路

遇到「某功能不生效」时的排查顺序：

```
1. 加日志确认代码执行到了吗（Log.e 关键节点）
2. 确认系统层权限（dumpsys accessibility / dumpsys package）
3. 确认事件真的收到了吗（onKeyEvent 里打 keycode）
4. 确认系统没拦截（return true 还是 false）
5. 确认视图真的加上了（addView 是否抛异常）
```

本项目 80% 的问题都是第 2 步——**权限和配置被系统重置**，而不是代码写错。
