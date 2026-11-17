# lsp_translation

把编辑器里 hover 弹出的英文文档翻译成中文的 LSP 代理。不用改编辑器、不用装扩展 ——
它就是一个夹在编辑器和真实语言服务器之间的 stdio 中间人。

```
编辑器 ──stdio──> lsp_translation ──stdio──> rust-analyzer / pyright / gopls / ...
                        │
                        └─ 只拦截 textDocument/hover 等响应，
                           把 markdown 文档翻成中文，其余字节原样透传
```

## 为什么是代理，不是 Zed 扩展

Zed / Wu 的扩展 API 没有 hover 钩子。`zed_extension_api` 里能改 LSP 行为的只有
`language_server_command`、`language_server_initialization_options`、
`language_server_workspace_configuration`、`label_for_completion`、`label_for_symbol`
这几项 —— hover 的返回内容不在其中。所以想改 hover 文本，只能在编辑器之下动手术。

好消息是 Wu / Zed 支持用 `lsp.<server>.binary.path` + `arguments` 完整覆盖启动命令
（`crates/project/src/lsp_store.rs`），所以把编辑器指向这个代理、再由代理拉起真正的
语言服务器，就能在不动编辑器源码的前提下改掉每一项 hover 结果。

## 编译

```bash
cargo build --release
```

产物 `target/release/lsp_translation.exe`（Windows 约 2.8 MB，静态链接区无额外依赖）。
Linux / WSL 版本见下面「WSL 项目」一节。

## 配置

### Wu / Zed

先把二进制放到家目录下的固定位置（Windows 放 `%USERPROFILE%\.local\bin\lsp_translation.exe`，
WSL 里放 `~/.local/bin/lsp_translation`），然后改 `settings.json`：

```jsonc
{
  "lsp": {
    "rust-analyzer": {
      "binary": {
        "path": "~/.local/bin/lsp_translation",
        "arguments": ["--", "rust-analyzer"]
      }
    }
  }
}
```

这五行同时管 Windows 项目和 WSL 项目 —— 见下面「WSL 项目」一节。

要点：

- `path` 支持 `~/` 前缀，由 `resolve_relative_path` 展开成各自平台的家目录
  （`crates/worktree/src/worktree.rs:2935`）。不带 `~/` 的相对路径会拼到项目根下，
  绝对路径则原样使用。
- 真正的语言服务器写在 `--` 之后。这里用**裸命令名**，靠 `PATH` 找 ——
  代理启动时会继承编辑器给它的 `shell_env`（登录 shell 的环境变量）。
- Windows 上 Rust 的 `Command` 会自动补 `.exe`，所以 `path` 不写扩展名也能跑。
- `arguments` 会整体替换该语言服务器原有的启动参数，所以别漏掉
  `--stdio`（pyright、gopls 这类需要显式声明的）：
  `["--", "pyright-langserver", "--stdio"]`。
- 原来的 `initialization_options` / `settings` 保持不动，代理原样转发。

改完重启编辑器；用 `zed: open log` 看有没有 `failed to spawn language server`。

### 换到别的语言

结构一模一样，只有两处要改：**settings 的 key 是这个语言服务器的名字**（不是语言名），
**`--` 后面原样照抄它原本的启动命令**。

第二条是关键 —— `arguments` 是**整体替换**，不是追加：

```rust
// crates/project/src/lsp_store.rs:950-953
if let Some(settings) = settings.binary.as_ref() {
    if let Some(arguments) = &settings.arguments {
        binary.arguments = arguments.iter().map(Into::into).collect();  // 直接覆盖
    }
}
```

而且一旦写了 `binary.path`，Zed 就短路返回，不再去 PATH 找、也不去下载
（`lsp_store.rs:801-843`）—— 所以 `ignore_system_version` 在这里没有意义。

| 语言 | settings key | arguments |
|------|--------------|-----------|
| Rust | `rust-analyzer` | `["--", "rust-analyzer"]` |
| Zig | `zls` | `["--", "zls"]` |
| Python (ty) | `ty` | `["--", "ty", "server"]` |
| C / C++ | `clangd` | `["--", "clangd"]` |
| Go | `gopls` | `["--", "gopls", "-mode=stdio"]` |
| Python (basedpyright) | `basedpyright` | `["--", "basedpyright-langserver", "--stdio"]` |
| Python (pyright) | `pyright` | `["--", "pyright-langserver", "--stdio"]` |
| Python (ruff) | `ruff` | `["--", "ruff", "server"]` |
| Python (pylsp) | `pylsp` | `["--", "pylsp"]` |
| TypeScript / JS | `vtsls` | `["--", "node", "<...>/vtsls.js", "--stdio"]` |
| JSON | `json-language-server` | `["--", "node", "<...>/vscode-json-language-server", "--stdio"]` |
| CSS | `vscode-css-language-server` | `["--", "node", "<...>/vscode-css-language-server", "--stdio"]` |
| YAML | `yaml-language-server` | `["--", "yaml-language-server", "--stdio"]` |
| Bash | `bash-language-server` | `["--", "bash-language-server", "start"]` |
| Tailwind | `tailwindcss-language-server` | `["--", "node", "<...>/tailwindcss-language-server", "--stdio"]` |

依据：`crates/languages/src/{rust.rs:1454, c.rs:420, go.rs:48, python.rs:351-500/645-810, vtsls.rs:29, json.rs:135, css.rs:22, yaml.rs:24, bash.rs:59, tailwind.rs:27, eslint.rs:24}`。

**Zig 有点特殊**：`zls` 不是内置适配器，是 `zig` 扩展提供的
（`%LOCALAPPDATA%\Wu\extensions\installed\zig\extension.toml` 里
`[language_servers.zls]`）。扩展自己会读 `LspSettings::for_worktree("zls")`
的 `binary.path`，但它读取之前，Zed 的通用短路（上面那段 `lsp_store.rs:801`）
已经先命中了，所以配置方式跟内置适配器完全一致。zls 不需要任何启动参数：

```jsonc
"zls": {
  "binary": { "path": "~/.local/bin/lsp_translation", "arguments": ["--", "zls"] },
  "settings": { "zls": { "enable_build_on_save": true } }   // 原有设置照旧
}
```

**Python 的 ty 是 opt-in 的** —— `crates/languages/src/python.rs:377`
`is_opt_in_for` 只对 `Python` 返回 true，而 `assets/settings/default.json:2600`
的默认值是 `["basedpyright", "ruff", "!ty", ...]`，所以必须先打开它：

```jsonc
"languages": {
  "Python": {
    "language_servers": ["ty"],
    "format_on_save": "on"
  }
},
"lsp": {
  "ty": {
    "binary": { "path": "~/.local/bin/lsp_translation", "arguments": ["--", "ty", "server"] }
  }
}
```

如果只配 `lsp.ty` 而不改 `languages.Python.language_servers`，ty 根本不会启动，
hover 还是英文 —— 这时候 `--log-tail` 一条记录都没有，很容易误判成代理坏了。

一个语言挂多个服务器时（Python 默认就是 basedpyright + ruff），**出 hover 的是哪个
就包哪个**。看不到中文先查 `--log-tail`：没记录说明是另一个 server 在出文档。

### WSL 项目

WSL 项目和 Windows 项目是**两套进程**：连接 WSL 时编辑器会把
`zed-remote-server` 拉进发行版里跑，LSP 也在那一侧启动
（`crates/remote_server/src/headless_project.rs:232` 建的是 `LspStore::new_local`），
客户端这边只负责显示。所以：

1. **代理必须编成 Linux 版放进 WSL 家目录**，不能指望 WSL 去跑那个 `.exe`。
   Windows 的 exe 靠 interop 确实能在 WSL 里被执行，但那是多绕一层、还依赖
   `binfmt_misc` 开着，没必要。
2. **`arguments` 里也必须是 Linux 的 rust-analyzer**。WSL 的 rust-analyzer 通常
   在 `~/.rustup/toolchains/*/bin/` 或 `/usr/lib/rustup/bin/`，靠登录 shell 的
   `PATH` 就能找到 —— 所以裸写 `rust-analyzer` 是对的。
3. 用户级 `settings.json` 会同步到 WSL 侧的 remote server
   （`crates/project/src/project_settings.rs` 的 `UpdateUserSettings`），
   所以一份配置两边生效。前提是不能写死 `D:/...` 这种 Windows 路径 ——
   在 WSL 里它会被当成相对路径拼到项目根下，然后启动失败。

在 WSL 里编 Linux 版：

```bash
# 在 WSL 里，项目拷到 ~ 下编（/mnt/d 上编会慢很多）
mkdir -p ~/lsp_translation_build
cd /path/to/lsp_translation && tar -c --exclude=target . | tar -x -C ~/lsp_translation_build
cd ~/lsp_translation_build && cargo build --release
mkdir -p ~/.local/bin && cp target/release/lsp_translation ~/.local/bin/
```

`ureq` 走 rustls + ring，静态链接进二进制，WSL 里不用装 OpenSSL。
三家翻译接口在 WSL 里都能直连（实测 Google 200、腾讯 200）。

> 如果你在 WSL 里遇到 `failed to spawn language server: No such file or directory`，
> 九成是 `PATH` 里没有 `rust-analyzer`。先 `which rust-analyzer` 确认；
> 没有就把 `arguments` 改成绝对路径，或在 `binary.env` 里补上 `PATH`：
> ```jsonc
> "env": { "PATH": "/home/you/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin:/usr/bin:/bin" }
> ```

### 其他编辑器

任何支持自定义 LSP 启动命令的编辑器都行（VS Code 的 `languageserver` 扩展、
Neovim 的 `cmd = {...}` 等），把命令换成
`lsp_translation -- <真实服务器>` 即可。

## 命令行

```
lsp_translation [选项] -- <语言服务器命令> [参数...]

  --target <lang>    目标语言，默认 zh-CN
  --timeout <secs>   单次翻译超时，默认 15
  --method <name>    要翻译的响应方法（可重复），默认:
                     textDocument/hover, textDocument/signatureHelp
  --no-signature     只翻 hover，不翻参数签名帮助
  --off              关掉翻译，退化为纯转发（对比/排障用）
  --print-config     打印生效的配置（key 打码）
  --list-providers   列出所有 provider 和模型，标出当前生效的
  --cache-list       列出缓存条目（模型、语言、原文摘要）
  --cache-clear      删掉缓存文件
  --init-config      把配置模板写到 ~/.config/lsp_translation/config.toml
  -h, --help
  -V, --version
```

`--cache-list` 的输出长这样，一眼能看出哪段文档是哪个模型翻的：

```
tokenhub/deepseek-v4-1-flash-economy-261017 zh-CN       1292字  alloc::macros
tokenhub/deepseek-v4-1-flash-economy-261017 zh-CN       9109字  alloc::string
tokenhub/deepseek-v4-1-flash-economy-261017 zh-CN        326字  alloc::string::String
```

遇到问题想确认是不是代理引起的，用 `LSP_TRANSLATE_DISABLE=1` 启动 —— 它会退化成
纯转发，行为应该和直连一模一样：

```jsonc
"arguments": ["--target", "zh-CN", "--", "rust-analyzer"]
```

排查具体哪次请求失败：

```bash
LSP_TRANSLATE_VERBOSE=1 lsp_translation -- rust-analyzer   # 日志走 stderr
```

## 翻译后端

只有一条路：**OpenAI 兼容的 `/chat/completions` 端点**，整篇文档一次送翻。

可以配任意多个 provider，每个 provider 下挂多个模型：

```toml
# 选哪个（不写就用第一个 provider 的第一个模型）
provider = "tokenhub"
model = "deepseek-v4-1-flash-economy-261017"

[[ai]]
name = "tokenhub"
endpoint = "https://ai-tokenhub.com/api/v1"   # 填到 /v1 为止，不要带 /chat/completions
key = "sk-..."
models = ["deepseek-v4-1-flash-economy-261017", "别的模型"]   # 第一个是默认
temperature = 0
max_tokens = 8000

[[ai]]
name = "ollama"
endpoint = "http://127.0.0.1:11434/v1"
key = "ollama"
models = ["qwen2.5-coder:7b"]
```

`model` 允许填清单以外的名字（临时试新模型不用改 `models`）。写了不存在的
`provider` 会退回第一个并在 stderr 打一行警告。`name` 省略的 provider 名字是
`default`。用 `--list-providers` 看当前有哪些可选。

环境变量仍然优先：`LSP_TRANSLATE_PROVIDER` / `LSP_TRANSLATE_MODEL`，老的
`LSP_TRANSLATE_AI_ENDPOINT` / `_KEY` / `_MODEL` 三件套会整体替换成一个叫 `env` 的 provider。

设计参考了 [FluentRead](https://github.com/FluentRead/FluentRead) 的
`src/providers/translation/ai-sdk/`，但**没有引 SDK**：它那边是
`createOpenAICompatible` + `generateText` 再把请求体 `transformRequestBody` 原样透传，
SDK 纯粹当 HTTP 客户端用。Rust 侧对应的 `async-openai` 会拖进整个 tokio 运行时，
2.8MB 的单文件要变成十几 MB，而在我们这三线程 stdio 模型里同步请求更简单。
40 行 `ureq` 就覆盖了请求、鉴权、自定义头、超时、截断检测。

从 FluentRead 抄过来的三个关键点：

| 要点 | FluentRead 出处 | 我们的做法 |
|------|----------------|-----------|
| 关掉思考链 | `modelThinking.ts:190-199` 按协议分派 | 统一 `reasoning_effort: "none"` |
| 文档保护靠遮罩，不靠模型自觉 | `documentLiterals.ts` / `documentMathProtection.ts` | `markdown::mask` 换 `{{n}}` 占位符 |
| 截断当失败 | —— | `finish_reason == "length"` 直接放弃 |

**为什么关思考**：deepseek 系默认先吐一大段 `reasoning_content`，实测 702 个 token
才开始输出正文，短文本延迟 9.3 秒。加 `reasoning_effort: "none"` 后 0.8 秒。
（`enable_thinking: false` 对这家网关无效，实测反而更慢。）

**延迟实测**（同一篇 2488 字符的 rustdoc）：

| 每块字符 | 块数 | 总耗时 |
|---------|------|--------|
| 600 | 5 | 2.5s |
| 900 | 3 | 3.0s |
| 1200 | 3 | 3.8s |
| 3000 | 1 | 6.4s |

模型是流式吐字的，单次请求耗时几乎正比于输出字符数。切成小块并行，总耗时由
最慢那块决定而不是全部之和。所以 `CHUNK_CHARS = 800`、`MAX_INFLIGHT = 6`。
再小质量会掉（上下文太碎，术语对不上）。

## 缓存

两级，键都带目标语言 + provider/模型（换语言、换模型都不会命中旧译文）：

- **内存 LRU**：4096 条，命中 0 成本。
- **磁盘**：`~/.cache/lsp_translation/hover.jsonl`。

**Windows 上也用 `~/.cache`**，不用 `%LOCALAPPDATA%`：

```
Windows   C:\Users\<你>\.cache\lsp_translation\hover.jsonl
WSL       /home/<你>/.cache/lsp_translation/hover.jsonl
```

两个平台结构一致，文档写一遍、排障时不用先想「我在哪个系统上」。
日志 `proxy.log` 放在同一个目录。两者都能用 `XDG_CACHE_HOME` 改到别处，
或者用 `cache.path` / `log.path` 单独指定。

磁盘是 JSONL，一行一条，字段自带说明：

```json
{"v":1,"target":"zh-CN","model":"tokenhub/deepseek-v4-1-flash-economy-261017",
 "src":"\n```rust\nalloc::vec::Vec\n```...","dst":"\n```rust\nalloc::vec::Vec\n```...","at":1794388455}
```

| 字段 | 含义 |
|------|------|
| `v` | 格式版本，将来改结构时好区分 |
| `target` / `model` | 目标语言、翻译用的 provider/模型 |
| `src` / `dst` | 原文、译文，JSON 转义所以换行原样存 |
| `at` | Unix 秒 |

为什么不是 TSV 了：旧格式是 `键<TAB>译文`，键里塞着 `\x01` 分隔符和转义过的全文，
打开就是一大团 `zh-CN\n```rust\n...`，既看不出对应哪个函数，也分不清是哪个模型翻的。

不用 sqlite/rmp 的原因：这里只有一个写者、量级几千条、写入是追加。JSONL 每条独立成行，
`jq`、`grep`、`tail -f`、手工编辑器都直接用得上。

读起来不方便时用 `--cache-list`（模型 / 语言 / 原文摘要），或 `--cache-clear` 清空。

每条写完立刻追加落盘，不攒批 —— 编辑器关掉时我们是跟着语言服务器一起被 SIGKILL 的，
`Drop` 根本不会跑，攒着的缓存全丢。追加一次几十微秒，而 hover 本身要几百毫秒。
攒到 256 条或超过容量时整份重写一遍（去掉同键的旧行）。

实测第二轮（新进程）：命中 1–3ms，冷启动 1.2–3.2s。

## 运行日志

hover 走缓存还是发请求，从编辑器里完全看不出来 —— 两种情况显示的都是中文。
所以每次 hover 都往 `~/.cache/lsp_translation/proxy.log` 记一行，用 `--log-tail` 翻：

```
$ lsp_translation --log-tail

2026-11-11 17:32:10 INFO  start: tokenhub/deepseek-v4-1-flash-economy-261017 -> https://... | target=zh-CN | cache=on
2026-11-11 17:32:22 INFO  hover 336字 cache=miss request=1chunk 1316ms alloc::vec::Vec
2026-11-11 17:32:24 INFO  hover 326字 cache=miss request=1chunk 1219ms alloc::string::String
2026-11-11 17:32:27 INFO  hover 9109字 cache=miss request=6chunk 3207ms alloc::string
2026-11-11 17:32:41 INFO  hover 336字 cache=disk 0ms alloc::vec::Vec
```

`cache=` 三档：`memory`（同进程内重复 hover）、`disk`（新进程命中，
0ms）、`miss`（真发了请求，后面跟着块数和耗时）。末尾是原文摘要 ——
hover 文档第一行通常是 `alloc::string::String` 这样的符号路径，比记哈希有用。

失败也会记，原因写在字段里：

```
hover 9109字 cache=miss discarded=structure(lines 19 -> 11) 3207ms alloc::string
hover 336字 cache=miss request=failed 8000ms alloc::vec::Vec
```

| 配置 | 效果 |
|------|------|
| `[log] level = "off"` | 不写日志 |
| `[log] level = "info"` | 默认，每次 hover 一行 |
| `[log] level = "debug"` | 再加分块数、占位符数量、每块请求细节 |
| `LSP_TRANSLATE_LOG=debug` | 环境变量覆盖级别 |
| `LSP_TRANSLATE_LOG_FILE=...` | 换个日志文件位置 |

日志不轮转：一行几十字节，hover 一天最多几百次，跑一年也就几 MB。
真嫌大了 `rm` 掉即可，写失败也只丢日志、不影响翻译。

`--log-path` 打印实际路径；`--print-config` 最后两行也能看到。

## 怎么保证代码不被翻坏

这是整个项目最麻烦的部分。做法和 FluentRead 的 `documentLiterals.ts` 一致 ——
**不指望模型自觉，而是让代码在物理上离开它的视野**：送翻前把不能动的东西换成
`{{0}}` `{{1}}`，模型连看都看不到，想改也没得改。

| 问题 | 处理 |
|------|------|
| ``` 围栏代码块整块被翻（`let x` → `让 x`） | 连围栏行一起换成一个占位符 |
| 行内代码 `` `foo` `` 被翻成「酒吧」 | 占位符遮住 |
| 链接 `[文字](url)` 的方括号被换成全角「【】」 | `](url)` 整段遮住，只留文字让模型翻 |
| 裸 URL 被改写 | 遮住 |
| `<code>` `<a href>` 标签被读成「小于 code 标签大于」 | 遮住 |
| 链接引用定义 `[x]: url` 整行都是结构 | 整行遮住 |
| 模型把占位符吃掉/重复 | `markdown::restore` 返回 None，整篇保留英文 |
| 模型多吐一个 `]`（`[文字](url)]`） | `repair_brackets` 删掉紧跟 `)` 的那个 |
| `finish_reason == "length"`（译文截断） | 整篇放弃，保留英文 |

占位符格式是 `{{0}}`、`{{1}}`… —— 对比过 8 种写法，只有这个在三个引擎上都能
原样保留（`[[0]]` 会被腾讯当日期模板乱翻，`⟦0⟧` 会被它整个吃掉）。个别情况下
引擎会往里面塞零宽字符，还原时按「只取数字」容错。

**还原后的二次校验**：占位符全部还回来之后，再核对每个围栏代码块的正文逐字
出现在结果里。这一步兜住的是「模型把占位符挪了位置」这类隐性损坏。

## 已知限制

- **只处理响应，不处理请求**。`textDocument/hover` 的 `contents` 支持
  Markdown / 纯文本两种，代码走 Markdown 分支。
- **首次 hover 有延迟**。短文档 1.5–2s，`String` 这种 6500 字的整篇 3.3s，之后走缓存
  1–3ms。编辑器侧 LSP 请求超时是 120 秒（`DEFAULT_LSP_REQUEST_TIMEOUT_SECS`），
  不会因为翻译卡死。
- **长文档切块并行**。800 字符一块、最多 6 个并发。块之间没有上下文，
  代词和术语偶尔会对不上（实测影响很小，rustdoc 句式本来就独立）。
- **翻译是异步的，响应可能乱序**。LSP 靠 `id` 匹配，乱序无害。
- **不翻译 `textDocument/completion` 的文档**。补全项数量多、单条短，逐条送翻会很慢且
  花不少 token，收益也低。想做的话改 `Options::methods` 并在 `translate_contents`
  里处理 `documentation` 字段即可。
- **只翻 AI，没有免费兜底**。没配 `[ai]` 就完全不翻（启动时会打一行提示）。
  想加回机器翻译接口的话，`markdown::mask` / `restore` 这两层可以直接复用。
- **表格只翻单元格文字**。zls 的 `| Base | Value |` / `| BIN | 0b1 |` 会变成
  `| 基数 | 值 |`，竖线和分隔行保留，但列宽不再对齐 —— Markdown 渲染出来一样。
- **代码块里的字符串字面量不翻**。这是有意的：`` ```zig\nfn add(a: u32, b: u32) u32\n``` ``
  必须原样，改了编辑器就认不出来。

## 测试

```bash
cargo nextest run        # 45 个单测，不需要联网
cargo clippy --all-targets
```

联调用自带的假服务器（返回一段写死的 Rust 文档）：

```bash
cargo build --release
python - <<'EOF'
import subprocess, json
p = subprocess.Popen(["target/release/lsp_translation.exe", "--",
                      "target/release/fake_lsp.exe"],
                     stdin=subprocess.PIPE, stdout=subprocess.PIPE, bufsize=0)
def frame(o):
    b = json.dumps(o).encode()
    return b"Content-Length: %d\r\n\r\n" % len(b) + b
p.stdin.write(b"".join(frame(m) for m in [
    {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"capabilities":{}}},
    {"jsonrpc":"2.0","method":"initialized","params":{}},
    {"jsonrpc":"2.0","id":2,"method":"textDocument/hover",
     "params":{"textDocument":{"uri":"file:///t.rs"},"position":{"line":3,"character":9}}},
])); p.stdin.flush()
# 读回响应看 contents.value
EOF
```

接真服务器的做法一样，把 `fake_lsp.exe` 换成 `rust-analyzer.exe`，多发一个
`textDocument/didOpen` 再等索引完成。
