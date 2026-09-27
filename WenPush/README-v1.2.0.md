# WenPush · GitHub 上传下载工作台

> 原生 Android 应用 · 双向文件传输
> 作者：青玄 | 版本：v1.1.0 | 包名：com.qingxuan.wenpush

## 这是什么

一个把手机文件跟 GitHub 仓库打通的小工具。**双向**：

- 上传：把手机里的文件 / 目录推送到仓库或 Release
- 下载：浏览仓库文件、查看 Release 资产，点一下就存到手机

任何人只要有 GitHub 账号就能用，不需要 Termux、不需要电脑、不需要 Python。

## 怎么开始用

1. 装 APK，打开，点“授权存储”给上文件权限
2. 去 GitHub 建一个 Personal Access Token：
   头像 → Settings → Developer settings → Personal access tokens
   勾 `repo` 权限（Fine-grained Token 就勾 Contents: Read and write）
3. 回到 App，底部配置区填 Token / 用户名 / 默认仓库，点保存
4. 上传页贴路径推送；下载页填仓库名浏览下载

## 上传：三条通道自动分流

| 体积 | 通道 | 机制 |
|------|------|------|
| 32MB 以下 | git | blob → tree → commit → ref 原子提交 |
| 32MB ~ 95MB | git-huge | 流式 base64，不吃内存 |
| 95MB 以上 | release | Releases 资产，单资产上限 2GB |
| 1.8GB 以上 | release | 自动分卷 .part001 |

同一仓库同一分支的提交自动串行，并发提交不会撞车。

## 下载：两条通道

- 仓库文件：走 contents API，支持目录逐级浏览、中文路径
- Release 资产：按 tag 分组列出，显示大小，一键下载
- 保存目录可自定义，默认 /sdcard/Download/WenPush

## 架构

    MainActivity.java   WebView 容器 + 权限 + SAF 选择 + JS 桥
    Bridge.java         任务队列 + 线程池 + 上传/下载调度
    Engine.java         上传引擎 + 下载引擎 + 并发锁
    GithubApi.java      REST 客户端（重试 + 流式传输）
    assets/index.html   单文件 UI（上传/下载双标签页）

设计要点：WebView + JS Bridge 替代本地 HTTP 服务，前端调 WenPush.xxx() 直进 Java，省掉 socket 层。

## 权限说明

- INTERNET：访问 GitHub API
- MANAGE_EXTERNAL_STORAGE：直接读写 /sdcard（建议授予）
- 不授予也能用：走系统文件选择器，App 拷到私有目录再操作

## 构建

    bash build.sh

依赖：aapt2 / d8 / zipalign / apksigner + android-23 平台 jar + JDK 21
产物：build/WenPush-v1.1.0.apk

签名：keystore `wenpush.keystore`，别名 `wenpush`，口令 `wenpush2026`，有效期 30 年

## 构建踩坑（重要）

1. build-tools 34.0.0 的 aapt2 是 x86_64 二进制，aarch64 上无法执行，必须用系统 /usr/bin/aapt2
2. aapt2 2.19 解析不了 android-35 资源表，必须用 android-23 平台 jar
3. android-23 不认 requestLegacyExternalStorage，已移除
4. android-23 无 Environment.isExternalStorageManager()，用反射调用
5. styles.xml 不能写裸色值，必须 @color 引用
6. aapt2 link 必须加 -A assets，否则 WebView 白屏
7. javac 失败被管道吞掉退出码会产出残缺 APK，已改 fail-fast

## 已验证

上传（v1.0.0 实测）：
- 单文件 / 目录递归 / 中文名 + 子目录 / 手机路径直推
- 100MB 走 Release，约 25 秒

下载（v1.1.0 实测）：
- contents 通道：HTTP 200，md5 与源文件逐字节一致
- Release 资产通道：跟随 302 后 HTTP 200，md5 一致
- 目录列举：返回文件列表正确

构建产物：
- 资源链接 / Java 编译（28 class）/ dex 生成 全部通过
- 签名 v1 + v2 + v3 三级验证通过
- assets/index.html 已打入（18846 字节）

## 未验证

- 真机安装运行（需手动点装，Shizuku 未运行无法自动安装）

---

青玄 · 2026-09-27
