---
name: git-doctor
description: '诊断并修正项目的 Git 配置问题，也负责按规范接入新配置。当前覆盖 .gitmodules 体检与子模块接入。Use when: auditing or fixing .gitmodules, diagnosing submodule clone failures, adding a git submodule, wiring a project to Project-Guidelines or its sub-repositories, setting up GitHub/Gitee mirror resolution; 用户要求检查 Git 配置、体检 .gitmodules、排查子模块拉取失败、添加子模块、配置 Gitee 镜像跳转。'
argument-hint: '留空则对当前仓库做体检；接入新配置时给出上游地址与挂载路径'
agent: agent
---

# 任务：Git 配置诊断与修正

一次性任务。先按分发表识别本次要执行的项目，**只执行命中的那一节**，完成后输出报告即结束。

诊断出的问题分两类处置：可安全修正的按对应章节直接修正；带取舍或涉及远端的按「输出」一节警告并给出建议命令，不擅自改写。

## 任务分发

| 任务 | 触发条件 | 执行章节 |
| :--- | :--- | :--- |
| `.gitmodules` 体检 | 未给出参数，或要求检查、修复、排查子模块问题 | 子模块：体检流程 |
| 接入子模块 | 给出了上游仓库地址 | 子模块：接入流程 |

未给出任何参数时默认执行体检，不要反问。无法在多个任务间唯一命中时，列出候选让用户选择，不要猜测意图；命中任务缺少必要参数（如挂载路径）时先向用户确认。

新增诊断项或 Git 任务时，在本表登记一行，并在下方增加对应的同级章节，不要把不同任务的步骤混进已有章节。

## 通用前置

先确认当前仓库属于哪一类，两类适用的约定不同：

```bash
git remote get-url origin
```

- **规范库**：`Project-Guidelines` 及其子模块仓库（`c-code-review`、`code-comment`、`prompts` 等）。
- **消费方项目**：引用规范库或其任一子库的其他项目。

## 子模块：约定依据

体检与接入两条流程共用以下三条约定，它们既是判定依据，也是修正目标。

### 约定 1：消费方以 HTTPS 方式引入（仅消费方）

消费方项目的子模块地址必须解析为 HTTPS。HTTPS 无需配置密钥，CI、容器与协作者可直接拉取。

两种情况需要警告：`.gitmodules` 中写死 SSH 地址（`git@<host>:...` 或 `ssh://...`）；或使用相对地址但父仓库 `origin` 为 SSH，解析后仍是 SSH。

**发现后只警告，不自动改写**，除非用户明确要求修改。

### 约定 2：优先相对地址，兼容 GitHub 与 Gitee 镜像（两类都适用）

相对地址按父仓库 `origin` 自动解析：从 GitHub 克隆走 GitHub，从 Gitee 克隆走 Gitee，平台网页上点击子模块也跳转同平台仓库，无需维护两份 `.gitmodules`。

| 场景 | 写法 | 父仓库 origin | 解析结果 |
| :--- | :--- | :--- | :--- |
| 子模块与父仓库同属主 | `../<repo>.git` | `https://github.com/Luyi365/<parent>.git` | `https://github.com/Luyi365/<repo>.git` |
| 同属主，Gitee 侧 | `../<repo>.git` | `https://gitee.com/Luyi365/<parent>.git` | `https://gitee.com/Luyi365/<repo>.git` |
| 跨属主引用 | `../../Luyi365/<repo>.git` | `https://github.com/<other>/<parent>.git` | `https://github.com/Luyi365/<repo>.git` |

前提：相对地址依赖父仓库存在 `origin` remote。以 ZIP 下载或无 remote 的仓库无法解析，子模块需另行获取。

Gitee 镜像不存在时保留绝对 HTTPS 地址（指向 GitHub），并提示用户补建镜像。此时**不要**改成相对地址，否则从 Gitee 克隆会因子模块仓库不存在而失败。

### 约定 3：消费方为每个子模块配置 ignore = dirty（仅消费方）

规范库只被消费方读取，不在消费方内改动，父仓库无需为子模块的工作区变化报警。

效果：父仓库的 `git status` 与 `git diff` 不再报告子模块工作区的改动与未跟踪文件，但仍报告子模块指向 commit 的变更，版本升级不会被隐藏。`git submodule status` 不受影响。需要查看被忽略的内容时用 `git status --ignore-submodules=none`。

规范库自身及其子库不加此配置，维护者需要看到子库的未提交改动。

## 子模块：接入流程

1. 确认上游地址与默认分支，并检索是否存在同名 Gitee 镜像：

   ```bash
   git ls-remote --symref https://github.com/<owner>/<repo>.git HEAD
   git ls-remote --symref https://gitee.com/<owner>/<repo>.git HEAD
   ```

   两侧都存在且 HEAD 指向同一 commit，才视为镜像可用。

2. 用显式 HTTPS 地址添加。相对地址在 `submodule add` 阶段会立即解析并克隆，先用绝对地址可避开本地协议与凭据问题：

   ```bash
   git submodule add https://github.com/<owner>/<repo>.git <path>
   ```

3. 镜像可用时，把 `.gitmodules` 中该条目的 `url` 改为相对地址，并同步到本地配置，否则仍按旧地址拉取：

   ```bash
   git submodule sync --recursive
   ```

4. 消费方项目为该条目补上 `ignore = dirty`。

5. 按「子模块：验证」确认，然后输出报告。

消费方项目的条目形如：

```ini
[submodule "rules/code-comment"]
	path = rules/code-comment
	url = ../code-comment.git
	ignore = dirty
```

## 子模块：体检流程

逐条检查 `.gitmodules` 中的每个子模块，按下表给出判定与处置。**直接修正**的当场改，**仅警告**的只报告并给出建议命令。

| 诊断项 | 判定方法 | 处置 |
| :--- | :--- | :--- |
| `path` 与实际目录不一致，或目录未作为 gitlink 记录在索引中 | 比对 `.gitmodules` 与 `git submodule status`、`git ls-files -s <path>` | 仅警告。修复涉及移动目录或改写索引，需用户决定 |
| 地址解析为 SSH | 见约定 1 的两种情况 | 仅警告。按「输出」的固定格式，不擅自改写 |
| 镜像可用却仍用绝对地址 | 两侧 `ls-remote` 都通且 HEAD 一致 | 直接修正为相对地址，并 `git submodule sync --recursive` |
| 镜像不存在却用了相对地址 | Gitee 侧 `ls-remote` 返回 404 | 仅警告。改回绝对 HTTPS 地址属于降级，需用户确认 |
| 镜像不存在 | 同上 | 仅提示用户补建镜像，不代为操作远端 |
| 消费方项目缺少 `ignore = dirty` | 见约定 3 | 直接补上。纯本地配置，可逆 |
| 规范库自身出现 `ignore = dirty` | 见约定 3 的例外 | 直接移除 |
| 父仓库锁定的 commit 在某一侧远端不可达 | 见下方命令 | 仅警告。多为 Gitee 尚未同步，等同步或让用户推送 |

最后一项的判定方式。Gitee 为单向同步，落后时会取不到父仓库刚记录的新 commit：

```bash
git -C <path> fetch https://gitee.com/<owner>/<repo>.git <branch>
git -C <path> merge-base --is-ancestor <父仓库锁定的 commit> FETCH_HEAD
```

有直接修正的改动时，改完按「子模块：验证」复核。

## 子模块：验证

```bash
# 子模块已检出，且锁定到预期 commit
git submodule status

# 相对地址在当前 origin 下解析正确
git config --get submodule.<path>.url
```

需要确认 Gitee 侧解析结果时，临时切换 `origin` 后重新 sync，**检查完毕立即还原**：

```bash
git remote set-url origin https://gitee.com/<owner>/<parent>.git
git submodule sync --recursive && git config --get submodule.<path>.url
git remote set-url origin <原地址> && git submodule sync --recursive
```

克隆消费方项目时一并拉取子模块：

```bash
git clone --recurse-submodules <parent-url>
git submodule update --init --recursive   # 已克隆过则补拉
```

## 输出

体检按子模块逐条给出判定，分三段汇总：**已修正**（做了什么改动）、**需人工处理**（警告项与建议命令）、**通过**。末尾给出优先处理清单。全部通过时明确说明「无异常」，不要堆无意义的确认。

接入报告：改动的文件、新增条目的最终内容、子模块锁定的 commit、两侧远端的验证结果、待用户处理的事项（如需补建的镜像）。

SSH 警告使用固定格式：

```text
⚠️ 子模块 `<path>` 解析为 SSH 地址：`<url>`
影响：缺少 SSH 密钥的环境（CI、新克隆、协作者）无法拉取该子模块。
建议：<具体命令>
已保留原配置，未自动修改。
```

## 收尾

- 除用户明确要求，不执行 `git commit`，只把改动留在工作区并说明改了什么。
- 临时改动过 `origin` 的，必须还原并复核 `git remote get-url origin`。
- 冲突时按此顺序取舍：目标仓库已有的强制约定与 CI 要求 → 子模块能在目标环境成功拉取 → 约定 2 的跨平台解析 → 约定 1 与约定 3（二者只警告或建议，不破坏前三项）。
