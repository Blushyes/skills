# Python CLI 参考（typer / click）—— 骨架

> 状态：骨架。`SKILL.md` 里的原则原样适用；只有 harness 不同。填到 `references/typescript.md`
> 的深度。结构与那份文件 1:1 对应。

## 1. 栈与配置
- **框架：** `typer`（类型注解驱动，建在 `click` 上）—— 要更多控制就直接用 `click`。两者都是主流；
  `argparse` 只在要零依赖时用。
- **当心启动税：** Python 的解释器 + import 成本是这四种语言里最高的，且在 *每次* agent spawn 时
  都付。让 import 懒加载（重依赖 `import` 放在命令 *内部*，别放模块顶部），并尽量单一入口模块。
- 测试：`pytest` + click 的 `CliRunner`（in-process）和/或 `subprocess`（真 e2e）。

## 2. 目录结构
```
mytool/
  __main__.py        # 薄入口：app()
  cli.py             # typer app + 全局 callback（--json/--format）
  output.py          # Report + emit（text | json | addr）
  errors.py          # CliError(Exception)，带 exit_code、next_steps
  domain/            # 纯核心：selector.py、ops/（registry + handlers）
  usecases/          # commands/<verb>.py（薄）、lib.py（resolve_targets、runner）
```

## 3–10. Harness —— TODO
- **CliError** → `class CliError(Exception)`，带 `exit_code`、`next_steps`；在顶层 callback 捕获、
  emit 后 `raise typer.Exit(code)`。（typer 把异常映射成退出码。）
- **Report + emit** → 一个 `@dataclass Report`（或 TypedDict）；`--json` 用 `json.dumps`；
  `--format addr` 打印 `rows[].addr`，且仅当 `rows` 存在时（§4 / 坑）。
- **dispatch** → typer 把 typed 参数变成 handler 签名；保持每个 `@app.command()` 函数体「校验 →
  委派 → emit」。
- **Option mixins** → typer 不像 clap/commander 那样好 flatten；实用做法是一个共享
  `selection_opts(...)` helper 返回一个 dataclass，或一个 click 装饰器栈（`@selection_options`）套到
  读+写命令上（§6 的想法）。
- **selector + resolve_targets** → 纯 `domain/selector.py`；`resolve_targets(doc, opts)` 共享。
- **op 引擎** → `REGISTRY: dict[str, Callable[[Ctx], None]]` + 一个 driver（§9）。在 `Ctx` 里传一个
  `run_ops` callable 来跑子 op，不引入模块 import 环。
- **verb-script** → 一个 quote-aware tokenizer（`shlex` *丢掉* 了 quoted/flag 的区分，所以 `do`
  surface 多半需要一个逐 token 记 `quoted` 的小自定义 tokenizer —— 坑 #1）；共享 template builder（坑 #4）。

## 11. 测试
- `CliRunner().invoke(app, [...])` 做快速 in-process 测试；`subprocess.run([...])` 跑真 argv/退出码
  （§11 理念）。`do -` 的 stdin 用 `input=` 喂。

## 12. 环检测
- Python *允许* import 环、且在 import 期咬人。用 `import-linter`（contracts）或 `pydeps`/`grimp`
  在 CI 里禁止 `domain.ops.* → driver`。经 `Ctx` 注入 re-entry。
