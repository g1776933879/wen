# WenPush Android 版

> GitHub 推送工作台 · 原生 Android 应用
> 作者：青玄 | 版本：v1.0.0 | 包名：com.qingxuan.wenpush

## 定位

把手机里的软件、文件、目录一键推送到 GitHub 的独立 App。
不依赖 Termux、不依赖 Python、不需要电脑。装上就能用。

## 架构

    MainActivity.java   WebView 容器 + 权限 + SAF 文件选择 + JS 桥
    Bridge.java         任务队列 + 线程池 + 状态推送
    Engine.java         推送引擎（三通道分流 + 原子提交 + 分卷）
    GithubApi.java      GitHub REST 客户端（重试 + 流式上传）
    assets/index.html   UI（单文件，复用已跑通的 Web 工作台）

关键设计：用 WebView + JS Bridge 替代本地 HTTP 服务。
前端调用 `WenPush.xxx()` 直接进 Java，省掉整个 socket 层。

## 三条通道

| 体积 | 通道 | 机制 |
|------|------|------|
| ≤32MB | git | blob→tree→commit→ref 原子提交 |
| 32~95MB | git-huge | 流式 base64 临时文件 |
| >95MB | release | Releases 资产（单资产上限 2GB）|
| >1.8GB | release | 自动分卷 .part001 |

## 权限

- INTERNET：访问 GitHub API
- MANAGE_EXTERNAL_STORAGE：直接读写 /sdcard（推荐授予）
- 未授予也可用：通过 SAF 选择文件，App 会拷到私有目录再推送

## 构建

    bash build.sh

依赖（已内置在环境里）：

    aapt2 / d8 / zipalign / apksigner   构建链
    android-23 平台 jar                  编译平台
    JDK 21                               javac

产物：`build/WenPush-v1.0.0.apk`

## 签名

    keystore   wenpush.keystore
    别名       wenpush
    口令       wenpush2026
    有效期     30 年

## 构建踩坑记录

1. **build-tools 34.0.0 的 aapt2 是 x86_64 二进制**，在 aarch64 上无法执行，
   必须用系统自带 aapt2（2.19）
2. **aapt2 2.19 解析不了 android-35 的资源表**
   （`RES_TABLE_TYPE_TYPE entry offsets overlap`），
   改用 `/usr/lib/android-sdk/platforms/android-23/android.jar`
3. **android-23 不认 `requestLegacyExternalStorage`**（API 29 属性），已移除
4. **android-23 无 `Environment.isExternalStorageManager()`**（API 30 方法），
   改用反射调用，避开编译期依赖
5. **样式里不能写裸色值**，必须 `@color/xxx` 引用
6. **aapt2 link 必须加 `-A assets`**，否则 WebView 白屏（这个最隐蔽）
7. **javac 失败被 `| head` 吞掉退出码**，导致产出残缺 APK，已改为 fail-fast

## 已验证

- [x] 资源编译链接通过
- [x] Java 编译通过（24 个 class）
- [x] dex 生成成功（20 个类全在）
- [x] assets/index.html 已打入（12449 字节）
- [x] 签名 v1 + v2 + v3 三级验证通过
- [x] manifest 元数据正确（minSdk 23 / targetSdk 34）

## 未验证

- [ ] 真机安装运行（需 Shizuku 或手动安装）

---

青玄 · 2026-09-27
