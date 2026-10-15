# steerings

本仓库是 `~/.pi/steerings/` 的版本库，保存代理使用的规范文件。`~/.pi/agent2/AGENTS.md` 是源文件；本仓库中的 `agent2-AGENTS.md` 是它的副本，由提交钩子生成。

## 目录

| 路径 | 说明 |
| --- | --- |
| 根目录下除 `README.md`、`agent2-AGENTS.md` 外的 `*.md` | 规范文件，直接编辑 |
| `agent2-AGENTS.md` | `~/.pi/agent2/AGENTS.md` 的副本，由钩子生成，禁止手工修改 |
| `.githooks/pre-commit` | 提交钩子，生成副本并纳入提交 |

## 首次设置

每个克隆执行一次：

```bash
git config core.hooksPath .githooks
```

## 日常用法

### 修改规范文件

照常执行 `git add` 与 `git commit`，无需额外操作。

### 修改 `~/.pi/agent2/AGENTS.md`

直接编辑源文件，禁止编辑 `agent2-AGENTS.md`。提交结果取决于暂存区与源文件是否有变化：

| 暂存区 | 源文件有变化 | 提交结果 | 你要做的 |
| --- | --- | --- | --- |
| 有内容 | 是 | 副本自动更新，并随本次提交入库 | 正常 `git commit` |
| 有内容 | 否 | 正常提交 | 正常 `git commit` |
| 为空 | 是 | 提交中止，提示副本已更新但未暂存 | `git add agent2-AGENTS.md`，再 `git commit` |
| 为空 | 否 | Git 报告无可提交内容 | 暂存需要提交的文件 |

源文件有变化且暂存区为空时，按下列步骤提交：

1. 执行 `git commit`，提交被中止，并提示 `agent2-AGENTS.md` 已更新但未暂存。
2. 执行 `git add agent2-AGENTS.md`。
3. 再次执行 `git commit`。

### 核对副本

在仓库根目录执行下列命令，无输出即副本与源文件一致：

```bash
diff <(sed -e 's|~/\.pi/steerings|.|g' ../agent2/AGENTS.md) <(tail -n +2 agent2-AGENTS.md)
```

## 常见提示

| 提示 | 原因 | 处理 |
| --- | --- | --- |
| `源文件不存在` | `~/.pi/agent2/AGENTS.md` 缺失或位置变化 | 恢复源文件，或修改 `.githooks/pre-commit` 中的 `src` |
| `副本中残留 steerings 绝对路径` | 源文件出现 `~/.pi/steerings/` 以外的写法，如 `$HOME/.pi/steerings` | 在源文件中统一写成 `~/.pi/steerings/` |
| `已更新但未暂存` | 暂存区为空，且副本有变化 | `git add agent2-AGENTS.md`，再 `git commit` |

## 注意事项

- `git commit --no-verify` 会跳过同步，副本可能与源文件不一致。
- 钩子通过相对路径 `../agent2/AGENTS.md` 读取源文件。移动本仓库或 `agent2` 目录后，需同步修改钩子。
