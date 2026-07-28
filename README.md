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
.
├── skills/                  # 所有可安装的 skill 都放在这里
│   └── <category>/          # 按主题分类的目录（如 design、dev、ops...）
│       └── <skill-name>/
│           └── SKILL.md     # 必需：带 frontmatter 的 skill 入口文件
├── templates/
│   └── SKILL.md.template    # 新建 skill 时可拷贝的模板
├── LICENSE
└── README.md
```

每个 skill 是 `skills/<category>/<skill-name>/` 下的一个目录，至少包含一个 `SKILL.md`，其 frontmatter 必须含 `name` 与 `description` 两个字段。

> **注意**：当前 `skills.sh` CLI 会发现本仓库的分类目录。如果仓库未来在根目录添加 `SKILL.md`，可使用 `--full-depth` 继续搜索所有子目录。也可直接使用子路径 URL 安装单个 skill：
>
> ```bash
> # 直接指向 skill 子路径（推荐）
> npx skills add https://github.com/Blushyes/skills/tree/main/skills/design/shadow-design
>
> # 如果根目录存在 SKILL.md，强制搜索所有子目录
> npx skills add Blushyes/skills --full-depth
> ```

## 新建一个 skill

```bash
# 方式 A：使用官方 CLI 生成模板
mkdir -p skills/<category>
cd skills/<category>/
npx skills init <skill-name>

# 方式 B：手动从模板拷贝
mkdir -p skills/<category>/<skill-name>
cp templates/SKILL.md.template skills/<category>/<skill-name>/SKILL.md
```

## 当前收录的 skills

<!-- skills-list:start -->

### 🎨 design

- [**shadow-design**](./skills/design/shadow-design/SKILL.md) - CSS 阴影/elevation 设计指南：四条铁律、5 种性格配方（uniform/sharp/diffuse/dreamy/floating）、完整 elevation token、dark mode 三方案、反模式清单

### 📊 data

- [**xquik-x-data**](./skills/data/xquik-x-data/SKILL.md) - Xquik 公开 X/Twitter 数据工作流指南，覆盖 REST API、SDK、MCP、webhook、搜索、账号资料、粉丝数据、趋势和 monitor route 选择

<!-- skills-list:end -->

## License

[MIT](./LICENSE)
