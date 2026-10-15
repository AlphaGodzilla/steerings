# 命令书写规范（pi-permission-system 兼容，推荐项）。

- **路径访问以 cwd 为界（单向包含）**：仅当被访问路径是当前工作目录本身或其**子孙**目录时自动放行；**父目录、兄弟目录、`~` 主目录均视为外部目录**（`external_directory` 默认 `"*": "ask"`），被命令引用即触发权限弹窗。例如 cwd 为 `~/.pi/agent` 时，`find ~/.pi ...` 会弹窗，而 `find . ...` 不会
- **一律使用相对路径**（相对于 cwd），不要书写 cwd 之外的绝对路径；确需访问 cwd 之外区域时，先征得用户同意（弹窗中按 `s` 批准整个会话）
- 禁止访问敏感目录（path 面 deny，用户无法批准）：`~/.ssh`、`~/.aws`、`~/.gnupg`、`~/.gitconfig`、`~/.m2`、`~/.npm`、`~/.kube`、`~/.docker`、`~/Documents`、`~/Downloads`、`~/Movies`、`~/Music`、`~/Pictures`、`~/.Trash`、`~/Library/Keychains`、`~/Library/Cookies`，以及任何 `.env`/密钥/凭证类文件
- 优先避免间接执行包装器（indirection wrapper）形式的命令，它们会强制触发权限确认弹窗且无法配置自动放行：
  `find ... -exec/-execdir/-ok/-okdir`、`fd ... -x/--exec`、`bash -c`、`sh -c`、`eval`、`sudo`、`env`、`xargs`、`nohup`、`timeout`、`watch` 等
- 遍历/批量读取文件时，优先用 `find <dir> -maxdepth N -type f -print`（纯列出）或 `grep -rn <pattern> <dir>` 定位，再用 read 工具读取，或对单个文件直接执行 `cat`/`sed -n '1,Np' <file>`/`head -n N <file>`（非包装器，正常放行）
- 环境变量优先直接写在命令里，而不是用 `env X=... cmd` 包装
- 以上均为推荐而非强制：若任务确实必须使用 wrapper 命令或访问外部目录（如 `find -exec`、`bash -c`、`find ~/.pi ...`），可以使用，但会触发权限确认弹窗（ask），等待用户批准即可；弹窗中按 `s` 可批准整个会话
