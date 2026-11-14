# skills

powercess 组织的 Agent Skills 集合。每个技能一个目录，遵循 [Agent Skills](https://code.claude.com/docs/en/skills) 约定：入口为 `SKILL.md`（YAML frontmatter 含 `name`、`description`），按需附带 `references/`、`agents/` 等附属文件。

## 技能列表

| 技能 | 说明 |
| --- | --- |
| [`bun-mode-manager`](skills/bun-mode-manager/SKILL.md) | 按项目需要设计或扩展 `bun run mode:*` 开发管理工具：统一运行入口、强制人类可读的彩色状态面板与分组帮助、本地自签 HTTPS、基于仓库身份的资源隔离；按需支持模式切换、容器增量构建与自定义任务。 |

## 安装

技能目录可以直接被支持 Agent Skills 的客户端读取。任选一种方式挂到本地技能目录（以 Claude Code / Codex 为例）：

```bash
# 全量克隆后软链单个技能
git clone https://github.com/powercess/skills ~/Projects/powercess-skills
ln -s ~/Projects/powercess-skills/skills/bun-mode-manager ~/.claude/skills/bun-mode-manager

# 或只取一个技能
mkdir -p /var/tmp/skill-fetch && cd /var/tmp/skill-fetch
git clone --depth 1 --filter=blob:none --sparse https://github.com/powercess/skills repo
cd repo && git sparse-checkout set skills/bun-mode-manager
```

## 新增技能

1. 在 `skills/` 下建立目录，写入 `SKILL.md`；frontmatter 的 `description` 说明**何时使用**，正文写流程与硬性约束。
2. 大段细节拆到 `references/`，在 `SKILL.md` 中用相对链接引用；不要写成单文件长文。
3. 需要按客户端自定义展示时放在 `agents/<client>.yaml`。
4. 更新上方技能列表后提交。
