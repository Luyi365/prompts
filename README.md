# prompts

按需调用的提示词集合。每个 `.prompt.md` 对应一个可手动触发的单次任务，由用户主动发起，执行完即结束。

本仓库是 [Project-Guidelines](https://github.com/Luyi365/Project-Guidelines) 的子模块，挂载在其 `prompts/` 目录下，也可以单独克隆使用。

## 提示词清单

| 提示词 | 调用 | 说明 |
| :----- | :-----: | :------ |
| [git-doctor](git-doctor.prompt.md) | `/git-doctor` | 诊断并修正 Git 配置。留空即对当前仓库做全量体检（`.gitmodules` 与行尾配置），也可指定单项检查或按规范接入子模块 |

## 安装与调用

格式遵循 [VS Code prompt file](https://code.visualstudio.com/docs/copilot/customization/prompt-files) 规范，支持该格式的 AI 工具均可使用。

- **安装**：复制到目标项目的 `.github/prompts/`（工作区级），或 VS Code 用户配置目录（跨工作区可用）
- **调用**：输入 `/` 加提示词名，如 `/git-doctor`；也可在命令面板运行 `Chat: Run Prompt`
- **传参**：在斜杠命令后补充信息，如 `/git-doctor 只检查行尾配置`

## 获取

```bash
git clone git@github.com:Luyi365/prompts.git        # GitHub
git clone https://gitee.com/Luyi365/prompts.git     # Gitee 镜像
```

供 AI 工具直接读取的 raw 链接：

| 提示词 | 爬取链接 | 国内镜像 |
| :-----: | :-----: | :-----: |
| git-doctor | [链接](https://raw.githubusercontent.com/Luyi365/prompts/refs/heads/main/git-doctor.prompt.md) | [链接](https://gitee.com/Luyi365/prompts/raw/main/git-doctor.prompt.md) |

Gitee 为单向同步，内容可能略滞后于 GitHub。

## 许可

[MIT](LICENSE)
