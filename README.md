# FUCK GOOGLE · 自愿安装

一个单文件 HTML，用 Google 没跟你打过招呼就装进硬盘的那个端侧模型来聊天。
零后端、零上传、零账号、零弹窗 —— 最后这一项，它输得很彻底。

在线地址：<https://mannixwu.github.io/FuckChromeAI/>（GitHub Pages，push 即自动重新部署）

授权：**MIT —— 看、改、再发布、商用都可以，保留版权声明即可**（详见文末）。

> 这个项目不是做一个更好的聊天工具。它唯一的论据是：**那个几百 MB 的模型真的在你硬盘上，而且此刻正在回答你的问题。**
> 页面上所有「查不到」的东西都写着「查不到」。为了好笑而编数字，会让这条论据变成一句废话。

---

## 30 秒跑起来

```bash
node serve.js            # → http://127.0.0.1:5321/index.html
```

也可以直接双击 `index.html` —— `file://` 属于安全上下文，API 照样暴露。
`serve.js` 只是本地实测用的静态服务器（`Cache-Control: no-store`，绑 `127.0.0.1`），可以扔。

必须在 **https / localhost / file** 下打开。局域网 `http://192.168.x.x` 里 `LanguageModel` 根本不存在，
那种情况下页面显示的「不支持」是安全上下文的问题，不是你的设备不行 —— 拦截页上会把这一条单独写出来。

## 前置条件（Google 文档写的，不是我编的）

| 项目 | 要求 |
| --- | --- |
| 浏览器 | 桌面版 Chrome 138+（实测在 155.0.8059.39 / Windows 10 上跑通） |
| Flags | `chrome://flags` 里把 `#optimization-guide-on-device-model` 和 `#prompt-api-for-gemini-nano` 设为 Enabled，重启 |
| 磁盘 | 存放 Chrome 个人资料的卷至少 **22 GB** 可用；下完之后可用空间跌破 **10 GB**，模型会被删掉，等空间够了再自己重下 |
| 企业策略 | `BuiltInAIDisallowed` 能把整台机器这条路焊死 |
| 查真实体积和下载进度 | `chrome://on-device-internals` |
| 移动端 | Android 版 Chrome、iOS 版 Chrome、非 Chromebook Plus 的 ChromeOS **不支持基础模型 API** —— 手机上看不到它，不是你没开 |

出处：<https://developer.chrome.com/docs/ai/prompt-api>

## 界面上有什么

- 多会话侧栏：新建 / 切换 / 双击重命名 / 删除，历史全部落在本地
- 流式输出，rAF 节流重绘；停止生成按钮；静默期显示阶段状态（追加还是重建会话 → 等首字 → 已经等了 N 秒）
- 自带 markdown 渲染：代码块（带复制）、表格、嵌套列表、行内代码、删除线、链接
- 每条回答可复制、最后一条可重新生成；空状态有建议提问 chip
- 参数：系统提示词、temperature 0–2、maxTokens 128–4096、历史上下文预算 512–8192 估算 token、自动标题开关
- 首轮问答后自动生成会话标题：本地先占位（关键词命中 emoji + 截断），再用模型后台改写一次
- 暗 / 亮主题，清空全部，以及一封「Google 口吻的公开信」
- 设备不支持时：**原界面一秒都不露脸**，全屏只有一张「恭喜，这次它没能塞进来」，附开启步骤与移动端说明

## 实测数字（Chrome 155.0.8059.39 / Windows，用户真实配置文件）

| 测的东西 | 结果 |
| --- | --- |
| `session.append()` 增量同步后出词 | **0.2 ~ 2.1s** |
| 丢弃 session、用完整历史 `create()` 重建 | **6 ~ 22.5s**（重新 prefill） |
| `create()` 同步 8 条历史 | 6.2s |
| 首字延迟 | 最长 **8.6s**；走 append 时低至 **168ms** |
| 生成速度 | 1997 字 / 44s ≈ **45 字/秒**（maxTokens=1024） |
| chunk 之间最长空档 | 81ms —— 所以「长答案看起来死机了」不是卡顿，是开头那十几秒界面只有一个空白光标 |
| markdown 渲染 | 13k 字整段渲染 0.6ms；8k 字逐帧灌入帧间隔 p50 10ms / max 13.4ms |
| 自动标题那次附加调用 | 临时 session **7.2s** 返回，之后 destroy；主会话第二轮仍是 append，首字 168ms |
| Chrome 自己列出的「受支持输出语言」 | `de / en / es / fr / ja` —— **没有 zh**。它照样能用中文写两千字，只是中文从来不在它保证质量的那份名单上 |

## 改代码之前请读这四条

1. **历史是真相，session 只是缓存。** localStorage 里的消息列表才是唯一数据源，session 一旦异常就从历史重建并重试一次。
2. **`mirroredIds` 前缀记账决定快慢。** 已同步进 session 的消息 id 按顺序比对，前缀对得上就 `append` 尾巴（毫秒级），对不上就整段重建（慢一到两个数量级）。记账漏一条 assistant 消息，症状直接表现为「慢 40 倍」。
3. **任何顺带调用模型的写法都必须另建临时 session 并 `destroy()`。** 起标题、探测、触发下载都是如此。复用主会话会让上面那套记账错位，代价由用户的下一次提问来付。
4. **后台只有一条 15 秒轮询**（`availability()`，只在 downloadable / downloading / unavailable 时问），而且这条轮询在页面上明写给了用户看 —— 因为整页骂的就是「不打招呼就去问」。别再偷偷加第二条。

新 API 表面（本机实测）：静态只有 `availability` / `create`，**没有 `params()`**；session 有
`prompt / promptStreaming / append / clone / destroy / contextWindow / contextUsage / measureContextUsage / oncontextoverflow`。
`Writer` / `Rewriter` / `assistant` 不存在。代码在 `create()` 抛 `TypeError` 时自动退回旧版 `initialSystemPrompt` + 整段历史重放。

## 开发用 URL 参数

| 参数 | 作用 |
| --- | --- |
| `?fake=1` | 桩模式：不调用任何模型，用假流式回一段覆盖渲染场景的文本。localStorage 命名空间隔离成 `fcaStub:*`，不污染真实会话 |
| `?gate=missing\|unavailable\|downloadable\|downloading\|unknown` | 强制演示拦截页的某种状态；`downloading` 的进度条会明确标注「?gate 演示，非真实进度」 |

## 你的数据去了哪里

哪儿都没去。全部在你浏览器的 localStorage 里，键名：`fca:convs:v1`、`fca:active`、`fca:settings`、`fca:theme`
（桩模式下前缀换成 `fcaStub:`）。侧栏「清空全部」即彻底删除，没有云端可留备份 —— 这正是重点。

账单面板里只有「本页面占用的存储」是能真实测量的，其余一律显示「无法得知 · 没有查询接口」。
那个「查不到」本身就是笑点，不要往里填假数字。

## 已知的不体面

- 安卓 / iOS 上它就是不能用，这条不由我实现决定。
- 刷新页面必然走一次重建（session 无法序列化），所以「重载后第一轮慢」是设计约束不是 bug。
- 长答案开头十几秒界面近乎静止，那是模型的节奏；现在会显示阶段状态，而不是让你对着光标猜。
- 手动改过名之后，自动标题就此闭嘴，不会覆盖你的命名。

## 授权

MIT，见 `LICENSE`。你可以拿去看、拿去改、拿去再发布、拿去商用，只要保留那份版权声明；
软件按「原样」提供，出了问题别找我（`index.html` 只有一个文件，出事的概率集中在你硬盘上那个几百 MB 的模型那边）。

不过这一条**写在前面**：想知道它到底对你的机器做了什么，把 `index.html` 从头读到尾就够了 ——
所有答案都在里面，包括那几行「无法得知 · 没有查询接口」。
这是本仓库唯一一条先声明再由你决定的规则，和某个几百 MB 的安装方式形成鲜明对比。

---

*本页面唯一能确切报出数字的东西，是它自己占的那点本地存储 —— 因为那是唯一一件它先问过你的。*
