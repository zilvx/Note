### ⚙️ CLI 命令：终端里的核心操作

这些命令直接在 macOS 终端里运行，用于启动、执行脚本或管理配置。

|命令|作用|常用示例|
|---|---|---|
|`opencode`|无参数启动，进入 TUI 交互界面[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)[](https://mintlify.wiki/opencode-ai/opencode/reference/commands)。|在项目目录下直接输入 `opencode`。|
|`opencode run`|**非交互模式**，直接执行一条指令，适合脚本和自动化任务[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)[](https://mintlify.wiki/anomalyco/opencode/cli/overview#param-print-logs)。|`opencode run "解释一下这个项目的入口文件"`[](https://developer.aliyun.com/article/1742766#1)  <br>`opencode run -m anthropic/claude-3.5-sonnet -f main.go "优化这段代码"`[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode agent`|管理自定义代理（Agent），比如创建或列出代理[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)。|`opencode agent create` (交互式创建)[](https://developer.aliyun.com/article/1742766#1)  <br>`opencode agent list` (列出所有代理)[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode auth`|管理 AI 提供商的 API 密钥和登录状态[](https://developer.aliyun.com/article/1742766#1)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)。|`opencode auth login --provider openai`[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode session`|管理所有会话，比如列出或删除历史会话[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)。|`opencode session list`[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)  <br>`opencode session delete <会话ID>`[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode models`|列出所有可用的 AI 模型[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)。|`opencode models --refresh` (刷新模型列表)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode stats`|查看 Token 使用量和费用统计[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)。|`opencode stats --days 7` (查看最近7天统计)[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/cli.md#1)|
|`opencode --help` / `-h`|查看任何命令的帮助文档。|`opencode run -h`[](https://developer.aliyun.com/article/1742766#1)|
|`opencode --version` / `-v`|查看当前 OpenCode 的版本号。|`opencode -v`[](https://developer.aliyun.com/article/1742766#1)[](https://mintlify.wiki/anomalyco/opencode/cli/overview#param-print-logs)|

### ⌨️ 斜杠指令：TUI 里的快捷操作

这些指令是在 TUI 界面中，输入 `/` 后跟指令名来使用的，用于控制会话[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。

|指令|别名|作用|快捷键 (前缀 `ctrl+x`)|
|---|---|---|---|
|**会话控制**||||
|`/new`|`/clear`|开始一个新的会话[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x n`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/sessions`|`/resume`, `/continue`|列出所有会话并在之间切换[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x l`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/compact`|`/summarize`|压缩当前会话的上下文，释放空间[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x c`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/undo`|-|**撤销**上一条消息及其文件更改[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|`ctrl+x u`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/redo`|-|**重做**被撤销的操作[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|`ctrl+x r`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/share`|-|生成当前会话的公开分享链接[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|`ctrl+x s`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|
|`/unshare`|-|撤销已分享的会话链接[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|-|
|`/export`|-|将当前对话导出为 Markdown 文件[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x x`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/exit`|`/quit`, `/q`|退出 OpenCode[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x q`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|**配置与显示**||||
|`/connect`|-|交互式地添加 AI 提供商和 API 密钥[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|-|
|`/models`|-|浏览并切换 AI 模型[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|`ctrl+x m`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/themes`|-|切换 TUI 的主题配色[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x t`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|`/thinking`|-|切换是否显示模型的推理过程[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|-|
|`/details`|-|切换是否显示工具执行的详细信息[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x d`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|
|`/editor`|-|打开外部编辑器（由 `$EDITOR` 定义）撰写长提示词[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|`ctrl+x e`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)|
|**项目与帮助**||||
|`/init`|-|创建或更新项目根目录下的 `AGENTS.md` 文件[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://developer.aliyun.com/article/1744828#1)[](https://mintlify.wiki/anomalyco/opencode/cli/commands/init#related-topics)|`ctrl+x i`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|
|`/help`|-|显示帮助对话框[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://github.com/sudo-tee/opencode.nvim#4)[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|`ctrl+x h`[](https://opencode.ai/docs/zh-tw/tui/#editor-setup#1)|

### 🧠 核心工作流：@explore, /plan 与 /build

这是 OpenCode 最实用的一套组合拳，专门用来理解和修改代码[](https://developer.aliyun.com/article/1744828#1)：

- **`@explore` (子代理)**：这是一个只读的“讲解员”，专门用于快速探索和回答关于代码库的问题，**永远不会修改任何文件**[](https://developer.aliyun.com/article/1744828#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。
    
    > 用法：`@explore 这个项目的认证逻辑是怎么实现的？`[](https://developer.aliyun.com/article/1744828#1)
    
- **`/plan` (代理模式)**：切换到 **Plan 模式**，它会分析代码并给出修改方案，但**禁止进行任何文件写入**，适合做安全的风险评估[](https://developer.aliyun.com/article/1744828#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。按 `Tab` 键可以从 Build 模式切换到 Plan 模式[](https://developer.aliyun.com/article/1744828#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。
    
- **`/build` (代理模式)**：这是**默认模式**，拥有完整的文件读写和命令执行权限，负责实际动手干活[](https://developer.aliyun.com/article/1744828#1)[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。
    

一个推荐的流程是：先用 `@explore` 读懂项目，再用 `/plan` 制定方案，最后切回 `/build` 执行[](https://developer.aliyun.com/article/1744828#1)。

### ✨ 进阶技巧

- **自定义斜杠指令**：你可以将常用的提示词保存为 `.md` 文件，放在 `~/.config/opencode/commands/` (全局) 或项目根目录的 `.opencode/commands/` 下，就能用 `/` 调用了[](https://github.com/wesammustafa/opencode-primer/blob/main/docs/reference/slash-commands.md)[](https://mintlify.wiki/anomalyco/opencode/cli/commands/init#related-topics)。
    
- **附加文件与执行命令**：在 TUI 输入框中，输入 `@` 可以模糊搜索并引用项目文件作为上下文，输入 `!` 可以执行 Shell 命令并将输出添加到对话中[](https://github.com/clouds-anr/opencode/blob/dev/OPENCODE_GUIDE.md#1)。