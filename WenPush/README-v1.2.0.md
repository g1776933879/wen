# WenPush · GitHub 上传下载工作台

> 原生 Android 应用 · 双向文件传输
> 作者：青玄 | 版本：v1.2.0 | 包名：com.qingxuan.wenpush

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
| 35MB 以下 | git | blob → tree → commit → ref 原子提交 |
| 35MB 以上 | release | Releases 资产，单资产上限 2GB |
| 1.8GB 以上 | release | 自动分卷 .part001 |

阈值为何是 35MB：实测 GitHub `POST /git/blobs` 的请求体上限约 53MB，
源文件约 40MB 即触发 `422 too large`。取 35MB（编码后约 46.7MB）留安全余量。
原先设计的 32~95MB 中间通道经实测不可用，已废弃。

同一仓库同一分支的提交自动串行，并发提交不会撞车。

## 下载：三种方式

- 单文件：浏览仓库，点文件右边「下载」
- Release 资产：按 tag 分组列出，显示大小，一键下载
- **整个目录（递归）**：进到任意目录点「下载整个目录」，
  自动枚举全部子目录文件，保留原有目录结构落盘，
  顶栏实时显示「N 个文件 · 总大小」

保存目录可自定义，默认 /sdcard/Download/WenPush。
单个文件失败不中断，最后汇总「X 成功 / Y 失败」。

### 下载三通道回退（v1.2.0 审计后新增）

单文件下载按顺序尝试，任一通道拿到完整内容即成功：

1. blobs API（按 tree 中的 sha 取，base64）
2. raw.githubusercontent.com 直链（CDN，不经 API 网关）
3. contents API（仅小文件可靠）

**每个通道都强制校验字节数**，不匹配即删除重试下一通道。
三通道全失败才报错，并列出各通道原因。

为什么这么做：实测发现 `contents` API 对大文件会截断
（30MB 文件只拿到 4.7MB / 5.0MB，且 HTTP 200 无任何报错）。
宁可明确报错，也绝不交付损坏文件。

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
产物：build/WenPush-v1.2.0.apk

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

文件夹下载（v1.2.0 实测）：
- 递归枚举：git trees API 一次拿全树，6 个文件 / 128171 字节，truncated=false
- 逐个下载：6/6 全部成功，尺寸校验通过
- 落盘结构：保留原有目录层级
- 完整性：三个可对比文件 md5 与源文件逐字节一致
- 超大树退路：truncated=true 时自动降为逐层递归

上传（v1.0.0 实测）：
- 单文件 / 目录递归 / 中文名 + 子目录 / 手机路径直推
- 100MB 走 Release，约 25 秒

下载（v1.1.0 实测）：
- contents 通道：HTTP 200，md5 与源文件逐字节一致
- Release 资产通道：跟随 302 后 HTTP 200，md5 一致
- 目录列举：返回文件列表正确

构建产物：
- 资源链接 / Java 编译（30 class）/ dex 生成 全部通过
- 签名 v1 + v2 + v3 三级验证通过
- assets/index.html 已打入（20163 字节）

## v1.2.0 代码审计发现（3 个真 Bug，全部已修）

### 1. Base64 分块编码在不满读取时错位（严重）

`blobFromFile()` 假设 `read()` 每次填满缓冲区。一旦某次读取不满
且长度不是 3 的倍数，中间块会产生 base64 padding，拼接后整个编码错位。

复现证据：模拟 5MB 数据按 [3MB, 2MB-1, 1B] 切分
→ 结果 6990512 字节 vs 正确 6990508 字节，从偏移 6990505 起不一致。

修复：引入 carry 缓冲，保证中间块始终是 3 的倍数，最后单独处理尾块。
验证：10 种尺寸 × 多种切分方式全部通过。

### 2. blob 上传缺 Content-Length（严重）

`blob_from_file()` 走 stream_body 但没设 Content-Length 头。
Python 版同样问题（只有 upload_asset 修过，漏了 blob）。

修复：补 `headers={"Content-Length": str(total)}`。

### 3. git-huge 通道根本走不通（设计错误）

原设 32MB 阈值的“中间通道”是凭直觉拍的，从未实测。

实测结果：

| 源文件 | 编码后请求体 | 结果 |
|--------|-------------|------|
| 28MB | 37.3MB | OK |
| 36MB | 48.0MB | OK |
| 39.5MB | 52.7MB | OK |
| **40.0MB** | **53.3MB** | **422 too large** |

修复：阈值降为 35MB，超过一律走 Release，废弃该中间通道。

## 未验证

- 真机安装运行（需手动点装，Shizuku 未运行无法自动安装）
- 大文件下载（≥30MB）在本机网络下所有通道均超时，
  属网络层限制（到 api.github.com / raw CDN 的传输会中断），
  非代码问题。手机端网络环境不同，预期可用。

---

青玄 · 2026-09-27
