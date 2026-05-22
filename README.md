# skills

[![skills.sh](https://skills.sh/b/Blushyes/skills)](https://skills.sh/Blushyes/skills)

Pan（[@Blushyes](https://github.com/Blushyes)）的个人 agent skills 合集，遵循 [skills.sh](https://skills.sh) 的开放生态格式，可被 Claude Code、OpenCode、Codex、Cursor 等 51+ agent 安装使用。

## 安装

```bash
# 安装本仓库全部 skill
npx skills add Blushyes/skills

# 列出可用 skill
npx skills add Blushyes/skills --list

# 只安装某一个 skill
npx skills add Blushyes/skills --skill <skill-name>

# 安装到全局（跨项目可用）
npx skills add Blushyes/skills -g

# 指定目标 agent
npx skills add Blushyes/skills -a claude-code
```

## 仓库结构

```
skills/
├── skills/             # 所有可安装的 skill 都放在这里
│   └── <skill-name>/
│       └── SKILL.md    # 必需：带 frontmatter 的 skill 入口文件
├── templates/
│   └── SKILL.md.template  # 新建 skill 时可拷贝的模板
├── LICENSE
└── README.md
```

每个 skill 是 `skills/<skill-name>/` 下的一个目录，至少包含一个 `SKILL.md`，其 frontmatter 必须含 `name` 与 `description` 两个字段。

## 新建一个 skill

```bash
# 方式 A：使用官方 CLI 生成模板
cd skills/
npx skills init <skill-name>

# 方式 B：手动从模板拷贝
cp -r templates/SKILL.md.template skills/<skill-name>/SKILL.md
```

## 当前收录的 skills

<!-- skills-list:start -->

骨架阶段，逐步添加中。

<!-- skills-list:end -->

## License

[MIT](./LICENSE)
