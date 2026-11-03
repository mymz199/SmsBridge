<details open> <summary><b>📚 目录（点击展开 / 收起）</b></summary>
一、两种传输方式

二、项目结构

三、使用步骤

四、协议与安全

五、免费更新（GitHub Releases，零成本）

六、编译

七、常见问题

八、Windows 接收端（从此版本开始提供）

九、声明

</details>
一个 最低支持 Android 4.4（API 19） 的短信转发 App：一台手机收短信，免注册地加密转发到另一台手机展示。

默认走 公共 MQTT 中继：两端只要能上网、填同一个「通信密钥」就能互通，不用注册、不用账号、不用买服务器。

也可以切到 局域网直连（TCP）：同一 Wi-Fi 直接连，不经过任何第三方服务器。

纯 Java + 系统 API（唯一依赖是 MQTT 客户端库），不使用 AndroidX，Android 4.4 可正常运行。

两种角色：转发端（插 SIM 卡、收短信）与 接收端（接收并展示）。

端到端加密：PBKDF2 + AES-256-GCM + HMAC 握手，中继服务器只能看到密文。

对方离线也能收到：中继使用 MQTT 持久会话，Broker 会为离线设备排队，重新上线后自动补发；发送端同时按 30s → 10min 兜底重试，直到收到 ACK。

支持长短信自动拼接、重复短信去重、开机自启、前台服务保活、接收端通知、可选「用本机短信再转发给另一个号码」。

内置本机自检和本机收发演示：一台设备就能验证「加密 → 协议 → 中继连通性」。

主页显示密钥指纹和对端在线状态，两端指纹一致才可能互通。

日志页可一键「复制诊断」，排查问题时直接把内容发出来即可。

系统拦截第三方短信广播时（MIUI / HyperOS、ColorOS 等），可用 Shizuku（ADB/root） 直接读取短信库。

首次启动必须阅读并勾选同意《使用须知与免责声明》，否则无法进入；设置里也可随时查看。

每个页面都有显式的「返回」按钮，标题栏也有返回箭头，没有返回键的设备也能正常退出。

[!TIP]
从本版本开始提供 Windows 接收端：Starting from this version, a Windows client is available. 详见下文 “八、Windows 接收端”。

🌐 一、两种传输方式
方式	需要什么	适合
公共中继（默认）	两端都能上网 + 同一个通信密钥	异地使用、零配置、零注册
局域网直连	两台设备在同一 Wi-Fi	不想走第三方服务器
text
                   ┌──────────────────────────────┐
   转发端 App  ───▶ │  公共 MQTT Broker (1883)      │ ◀───  接收端 App
   发布 /msg        │  主题由密钥 SHA-256 派生       │      订阅 /msg，回 /ack
                   └──────────────────────────────┘
                    只能看到 AES-GCM 密文
主题：smsbridge/<sha256(通信密钥) 前 24 位>/msg 与 /ack，别人猜不到。

负载：SMSB 帧 + AES-256-GCM 密文（与 TCP 模式完全相同的协议）。

发送端 QoS1 发布，接收端解密后回 ACK；发送端收到 ACK 才标记「已送达」。

对方离线时消息留在本地队列，恢复后自动重发（不会因为中继不留存而丢消息）。

常用公共 Broker（都不需要注册）
地址	说明
broker-cn.emqx.io:1883	默认，国内访问较快
broker.emqx.io:1883	全球节点
broker.hivemq.com:1883	备用
[!NOTE]
本项目的 Relay 代码已在电脑上对上面三个 Broker 做过真实收发测试：发布 → 接收 → 解密 → 回 ACK 全链路通过。

另外实测了离线场景：接收端下线 → 发送端发布 → 接收端重新上线 → 补发成功 → 回 ACK，完整通过。

🔁 离线补发（重点）
中继使用 MQTT 持久会话（cleanSession=false）+ QoS1：

接收端断网、退出或手机重启时，Broker 会把消息留在它的会话队列里；

接收端重新上线后，Broker 自动补发，不需要对方一直在后台挂着；

发送端本地也保留队列，按 30s → 10min 重试，直到收到接收端的 ACK；

即使超过 Broker 的会话保留时间，发送端仍会把未确认的消息重新发一遍。

结论：所以「离线后上线也能收到」是 Broker 排队 + 应用层重试 的双保险。

设置页已按 ① 通信密钥 / ② 传输方式 / ③ 短信读取 / ④ 运行与通知 / ⑤ 维护与自检 分组。

🗂️ 二、项目结构
text
smsbridge/
├── build.gradle / settings.gradle / gradle.properties
├── .github/workflows/android.yml          # 云端自动打 APK
└── app/
    ├── build.gradle
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/com/kony/smsbridge/
        │   ├── App.java                   应用入口
        │   ├── Prefs.java                 配置
        │   ├── Db.java                    发件箱 / 收件箱（SQLite）
        │   ├── Msg.java                   消息模型
        │   ├── Crypto.java                PBKDF2 / AES-GCM / HMAC
        │   ├── Wire.java                  帧格式
        │   ├── Codec.java                 帧的封装/解封（纯 Java）
        │   ├── Relay.java                 公共 MQTT 中继（纯 Java）
        │   ├── Conn.java                  TCP 加密连接（直连模式）
        │   ├── NetServer.java             TCP 服务端（直连模式）
        │   ├── BridgeService.java         前台服务 / 收发调度
        │   ├── NetError.java              网络错误的中文解释
        │   ├── SelfTest.java              本机自检
        │   ├── SmsReceiver.java           系统短信广播
        │   ├── BootReceiver.java          开机自启
        │   ├── MainActivity.java          主页 / 状态 / 说明
        │   ├── RoleActivity.java          首次进入选择模式
        │   ├── SettingsActivity.java      设置
        │   ├── InboxActivity.java         收件箱 / 发件箱
        │   └── LogActivity.java           运行日志
        └── res/                           布局、图标、主题
🚀 三、使用步骤
1. 两台设备各装一个 APK
第一次打开是两步：先选角色（转发端 / 接收端），再选短信读取方式（普通 / 加强）。
加强模式在 Android 6.0 以下会置灰并提示不可用；选完也能在「设置」里随时改。

设备	角色
插 SIM 卡的手机	转发端
随身手机 / 平板	接收端
2. 设置里填同一个通信密钥
传输方式选 公共中继（默认）；

点「随机生成」生成一个密钥，复制到另一端（或两端手输同一串）；

密钥越长越随机越好（默认生成 16 位十六进制）。

[!WARNING]
中继主题由密钥派生：密钥不同 = 主题不同 = 互相收不到；密钥相同 = 自动配对。两端密钥必须完全一致。

3. 两端都点「启动服务」
状态显示：

转发端：中继在线 · 等待对方确认

接收端：中继在线 · 等待新消息

之后转发端收到短信会自动加密发出，接收端落库并弹通知。

4. 自检（可选，但推荐）
主页点「本机自检」，会依次检查：

text
① 本机 TCP 监听 … 通过
② HMAC 握手认证 … 通过
③ AES-256-GCM 加密收发 + ACK … 通过
④ 错误密钥被拒绝 … 通过
⑤ 公共中继连通性 … 通过（往返 xxx ms）
[!NOTE]
第 ④ 步日志里的「握手失败」是预期结果。

5. 系统不给第三方发短信广播？用 Shizuku
很多国产 ROM 只把短信交给默认短信应用，第三方 App 收不到 SMS_RECEIVED。本应用支持用 Shizuku 以 ADB/root 身份直接读短信库：

安装并启动 Shizuku（root 或「无线调试」启动，要求 Android 6.0+）；

打开本应用「设置 → 短信读取」，选 Shizuku / ADB 读取（首次设置时也可以直接选「加强模式」）；

选中「加强模式」或点「授权 Shizuku 读取短信」时，会自动弹出 Shizuku 授权窗口，点「允许」即可；

保存（会自动重启服务）。

原理：Shizuku 让本应用以 shell(uid 2000) 或 root 身份执行
content query --uri content://sms/inbox --where "_id > 游标"。
AOSP 的 Shell 账号本身就持有 READ_SMS，所以系统拦了广播也照样能读。应用每 8 秒轮询一次，只转发游标之后的新短信。

[!TIP]
注意：Shizuku 每次开机都要重新启动（root 除外）；轮询会让耗电略增。

6. 局域网直连（不用公共中继）
设置 → 传输方式改成 局域网直连（TCP）：

接收端：「监听地址」填 0.0.0.0，启动服务；

转发端：「对端地址」填接收端的局域网 IP，端口保持一致；

两端仍用同一个通信密钥。

局域网直连走的是 App 内置的加密 TCP（协议与中继模式完全一致），不经过任何第三方。

🔐 四、协议与安全
text
帧:  'S' 'M' 'S' 'B' | ver(1) | type(1) | len(4) | body(len)

握手 (type=1/2/3, 明文, 仅 TCP 直连模式):
    client → server:  ts(8) | nonce(16) | HMAC-SHA256(key, "C"|ts|nonce)
    server → client:  ts(8) | nonce(16) | HMAC-SHA256(key, "S"|ts|nonce)
    时间偏差超过 5 分钟直接拒绝（防重放）

业务帧 (body 加密):
    body = nonce(12) | AES-256-GCM(plaintext, AAD = magic|ver|type)
    type=4 短信 JSON  {"v":1,"id","addr","body","ts"}
    type=5 ACK        "<消息ID>"
    type=6/7 心跳 PING/PONG（仅 TCP 模式）
密钥：PBKDF2WithHmacSHA1(通信密钥, "SmsBridge-v1", 20000 次, 256 位)；

中继模式下，Broker 只能看到：主题哈希、消息大小、发送时间、密文；看不到号码和内容；

短信在本地 SQLite 里是明文（应用私有目录），传输过程中是密文；

认证/解密失败会被静默丢弃，不会泄露任何信息；

提醒：公共 Broker 是免费测试服务，适合个人自用；对可用性要求高时请自建 MQTT（如 EMQX / Mosquitto），在设置里换地址即可。

🔄 五、免费更新（GitHub Releases，零成本）
App 内置了基于 GitHub Releases 的更新检查：不需要服务器、不需要账号，公开仓库完全免费。

text
App ──GET──▶ api.github.com/repos/<user>/<repo>/releases/latest
             │  比较 tag_name 与本地 versionName
             ▼
       发现新版本 → 下载 release 里的 .apk → content:// 交给系统安装器
启动时每 12 小时自动检查一次；「设置 → ⑤ 维护与自检 → 检查更新」可手动检查；

更新仓库在「设置 → ⑤」里填写，格式 用户名/仓库名，例如 yourname/SmsBridge（默认值只是占位）。

一次性准备
把工程推到一个公开仓库；

生成一个固定签名（以后所有版本都用它，否则手机拒绝覆盖安装）：

bash
keytool -genkeypair -v -keystore keystore.jks -alias smsbridge \
  -keyalg RSA -keysize 2048 -validity 10000
在仓库 Settings → Secrets and variables → Actions 添加：

KEYSTORE_BASE64：base64 -w0 keystore.jks 的输出

KEYSTORE_PASSWORD、KEY_ALIAS、KEY_PASSWORD

keystore.jks / keystore.properties 只留本地（已在 .gitignore）。

每次发版
bash
# 1. 改 app/build.gradle：versionCode +1，versionName 改成 2.5
# 2. 提交并打 tag
git commit -am "release 2.5"
git tag v2.5
git push origin master --tags
release.yml 会自动构建、签名、创建 Release，并把 SmsBridge-v2.5.apk 挂上去。
App 下次检查更新就能看到并一键下载安装。

[!WARNING]

签名必须一致，换签名会提示“应用未安装 / 签名不一致”；

用户需要允许本应用「安装未知应用」（Android 8+ 会引导去设置）；

GitHub 在国内可能较慢，下载失败可多试几次或换网络；

私有仓库需要 Token，本方案默认只支持公开仓库。

🛠️ 六、编译
方式 A：Android Studio
打开 smsbridge 目录，等待 Gradle 同步；

Build → Build Bundle(s)/APK(s) → Build APK(s)；

APK 在 app/build/outputs/apk/debug/app-debug.apk。

方式 B：命令行
需要 JDK 11+ 和 Android SDK（platforms;android-33、build-tools;33.0.2）。

bash
cd smsbridge
gradle assembleDebug
方式 C：GitHub Actions
仓库自带 .github/workflows/android.yml：Actions → build → Run workflow，完成后在 Artifacts 下载。

[!NOTE]
本工程已用 JDK 17 + Android SDK 33 + Gradle 7.6.4 实际编译并 lint 通过（0 error）。
可直接安装的调试包在项目根目录：SmsBridge-2.4-debug.apk（minSdk 19）。
工程在中文路径下也能构建（已加 android.overridePathCheck=true）；若个别机器 aapt2 仍报错，复制到纯英文路径再编译。

❓ 七、常见问题
Q：点「发一条测试短信」后切换到接收端，为什么收件箱里没有？
「测试短信」是放进发件箱、发给另一台设备的，本机切换角色不会把它搬到收件箱。
想在单机上看到完整效果，用主页的「本机收发演示（经中继）」：它会真的经公共中继往返一次，并按当前角色写进收件箱 / 发件箱。
另外「短信记录」从 1.5 起是收发合并的时间线：切换角色后依然能看到全部记录（带「收到 / 发出 · 已送达」标记）。

Q：两端都显示「中继在线」，但收不到短信？
先比对两端主页的密钥指纹：不一致就说明密钥不同（主题也不同），改成同一个密钥即可。

两端通信密钥是否完全一致（区分大小写、结尾不要有空格）；

转发端主页是否显示「已送达」；如果一直「待发送」，说明对端没回 ACK（多半是密钥不同）；

换一个中继服务器（broker.emqx.io / broker.hivemq.com）再试；

先跑一次「本机自检」看第 ⑤ 步是否通过。

Q：日志里刷 Connection refused / ECONNREFUSED？
这是 TCP 直连 模式才会出现的：说明本机端口上没有服务在监听。

接收端是否已启动（主页显示「监听中」）；

接收端监听地址是否为 0.0.0.0，转发端填的 IP 是否正确；

两台设备是否在同一 Wi-Fi。

Q：中继连接失败 / 超时？
公司网络、校园网或运营商可能封了 1883 端口。换 broker.emqx.io、broker.hivemq.com 试试；仍然不行就把传输方式改成「局域网直连」，或者自建一个 MQTT 服务器（1883 / 8883）。

Q：4.4 手机收不到短信广播？
确认已授予短信权限（6.0+ 需要运行时授权）。少数 ROM 需要把应用设为默认短信应用才会把 SMS_RECEIVED 发给第三方；本应用不申请成为默认短信应用，如遇此情况请换用原生系统的设备做转发端。

Q：Android 12+ 开机自启失败？
Android 12 起限制了后台启动前台服务，BOOT_COMPLETED 下 dataSync 类型可能被拒绝。打开一次 App 即可恢复；也可以在系统设置里允许「自启动 / 后台弹出界面」。

Q：耗电吗？
中继模式是一条 TLS 无关的 TCP 长连接 + 45s 心跳，待机功耗很低；「保持唤醒」会持有 PARTIAL_WAKE_LOCK 和 WifiLock，介意可在设置里关掉。

Q：会不会重复发送？
每条短信有稳定 ID（sha1(号码|时间|内容)），收件箱有唯一索引，重发会被去重；发件箱只有收到 ACK 才标记「已送达」。

🖥️ 八、Windows 接收端（从此版本开始提供）
[!TIP]
Starting from this version, a Windows client is available.

与 Android「短信桥」转发端协议完全一致的 Windows 接收端：只接收、展示、回执，不发送短信。适合把插 SIM 卡手机收到的短信同步到电脑。

单文件 SmsBridgeReceiver.exe，复制即可运行；无需安装 Python / Java / MQTT 库 / NuGet 包。

支持 公共中继（MQTT） 与 局域网直连（TCP） 两种方式。

收到短信后：落库 → 托盘弹窗 + 提示音 → 自动回 ACK（手机显示“已送达”）。

数据全部保存在本机；中继服务器只能看到主题哈希与密文。

详细说明见仓库中的 Windows 接收端 README.md。

运行要求
项目	要求
系统	Windows 7 SP1 / 8 / 8.1 / 10 / 11（32 位或 64 位）
运行时	不需要额外安装：Win7 自带 .NET Framework 3.5，Win8 及以上自带 4.x
依赖	无。SmsBridgeReceiver.exe 复制走就能运行
[!NOTE]
同目录的 SmsBridgeReceiver.exe.config 只是让 Win8+ 优先使用自带的 .NET 4.x；即使没有这个文件，exe 也能正常启动。

快速开始（公共中继，推荐）
手机打开原版 App，选择 转发端，在设置里生成/填写 通信密钥。

Windows 打开 SmsBridgeReceiver.exe，进入 设置，填入同一个通信密钥。

两端的「密钥指纹」必须完全一致（例如 B4D1-C6F8）。

传输方式保持 公共中继（MQTT），服务器默认 broker-cn.emqx.io:1883。

点右上角 启动接收。

显示「中继在线 · 等待对方上线」→ 说明 Windows 已连上中继；

手机也启动服务后，会变成「中继在线 · 对方在线」。

手机收到一条短信 → Windows 托盘弹出通知、收件箱出现记录，手机会显示「已送达」。

[!WARNING]
两端必须使用完全相同的密钥。密钥不同时，消息会被静默丢弃（与手机端行为一致）。

密钥建议直接点「随机生成」再「复制密钥」发给手机（16 位十六进制，纯 ASCII）。
部分 Android 版本对中文密钥的 PBKDF2 编码方式不同，中文密钥可能无法互通。

局域网直连（TCP，不经第三方）
传输方式选 局域网直连（TCP）。

默认由 接收端监听 0.0.0.0:8686：

手机转发端的「服务器地址」填 Windows 的局域网 IP（如 192.168.1.10），端口 8686；

Windows 防火墙首次会询问，请允许 专用网络。

也可以选 转发端监听：由 Windows 主动连接手机的 IP:8686。

两端仍需同一个通信密钥；两台设备的系统时间相差不能超过 5 分钟（握手会校验）。

<details> <summary><b>⚙️ 点击展开：Windows 端设置项摘要</b></summary>
设置	说明
通信密钥	与手机转发端一致；至少 4 个字符，建议点「随机生成」后复制到手机
公共中继	免注册，适合不同网络；默认 broker-cn.emqx.io:1883
局域网直连	同一 Wi-Fi 下延迟最低，不经过任何第三方
谁监听端口	接收端监听（默认）/ 转发端监听（本机去连接手机）
启动程序后自动开始接收	打开程序即自动连中继/监听
开机自启（当前用户）	写入 HKCU\...\Run，开机后最小化到托盘
关闭窗口时最小化到托盘	关闭按钮不退出，后台继续接收；右键托盘图标可退出
新短信弹窗 / 提示音	托盘气泡通知 + 系统提示音
保存并应用	保存设置；如果服务正在运行会自动重启服务
收件箱：双击查看全文，支持多选复制、删除、清空、搜索、导出 CSV。
运行日志：只记录号码和字数，不记录短信正文；「复制诊断」会带上系统信息、状态和最近日志（不含密钥）。

</details>
数据与隐私
所有数据都在：

text
%APPDATA%\SmsBridge\
  config.ini     配置（含通信密钥，请勿外传）
  messages.tsv   收件箱（Base64 字段，最多保留 2000 条）
  app.log        运行日志（不含短信正文）
中继服务器只能看到：主题哈希、密文长度、收发时间；看不到号码和内容。

密钥错误的消息无法解密，会被丢弃并记一条日志。

本程序只同步自己的短信，请勿用于侵犯他人隐私或任何违法用途。

<details> <summary><b>🧯 点击展开：Windows 端常见问题</b></summary>
一直显示「中继在线 · 等待对方上线」

手机端没有点「启动服务」；

两端密钥不一致（看指纹）；

手机和 Windows 用的不是同一个中继服务器；

公司/校园网络屏蔽了 1883 端口，换 broker.emqx.io 或 broker.hivemq.com 试试。

提示「通信密钥不一致」

两端密钥不同（区分大小写、不要有空格、不要复制到换行）。

TCP 连不上 / 连接超时

IP 填错、不在同一 Wi-Fi、Windows 防火墙拦截、端口被占用；

接收端监听时，手机填 Windows 的 IP；转发端监听时，Windows 填手机的 IP。

托盘图标不见了 / 再次打开提示「已经在运行」

程序在系统托盘里，双击图标恢复；右键 → 退出。

关闭窗口后还在接收

默认「关闭窗口时最小化到托盘」；要彻底退出请右键托盘图标 → 退出。

Win7 提示缺少 .NET Framework

极少数精简版系统删除了 .NET 3.5，安装 .NET Framework 4.8 即可（exe 会自动使用 4.x）。

</details>
本机自检
设置页 → 本机自检，会依次检查：

本机加解密往返（PBKDF2 + AES-256-GCM）

错误密钥被拒绝

本机 TCP 监听 / 握手 / 短信 / ACK

公共中继连通性（发布 → 中继 → 收回）

[!NOTE]
第 2 步显示「拒绝」是预期结果。自检不需要第二台设备。

从源码构建
powershell
powershell -ExecutionPolicy Bypass -File .\build.ps1
使用 Windows 自带的 csc.exe（.NET Framework 3.5），不需要 Visual Studio / dotnet SDK；

输出到 dist\SmsBridgeReceiver.exe；

源码在 src\，纯 C# 3 + WinForms，无第三方库。

[!IMPORTANT]
再次说明：本仓库不提供 Windows 端源码或 exe 的直接下载；如需使用，请从你获取到的发布包中自行提取。

与 Android 端的关系
Android 主项目提供转发端 / 接收端；Windows 端是新增的接收端，协议与 Android 转发端完全一致。

两端必须使用同一个通信密钥，且密钥指纹一致才可能互通。

Windows 端只接收、展示、回执，不发送短信。

中继主题、帧格式、AES-256-GCM、ACK 等与 Android 端一致，协议细节见 “四、协议与安全”。

📜 九、声明
本项目仅用于个人学习、研究与设备间同步自己的短信，请勿用于侵犯他人隐私或任何违法用途。

应用内《使用须知与免责声明》必须勾选同意后才能使用；修改声明内容时会要求重新同意。

公共 MQTT Broker 由各服务商提供，请遵守其使用条款；数据在到达 Broker 前已加密。

各公共 Broker 的会话保留时间不同（EMQX 默认约 2 小时），离线更久时由发送端的重试兜底，不会丢消息。

[!IMPORTANT]
源码与文件分发：本项目不提供在线源码托管，也不提供网盘 / Release 下载。仓库中只保留说明文档与协议描述；需要查看实现或自行构建的用户，请从你获取到的 APK / exe 安装包中自行反编译或提取。反编译仅限个人学习、研究与互操作性调试，请遵守当地法律法规及原软件许可；因自行反编译、修改、重打包产生的任何问题，与本项目作者无关。

