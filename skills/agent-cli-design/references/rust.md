# Rust CLI 参考（clap）—— 骨架

> 状态：骨架。`SKILL.md` 里的原则原样适用；只有 harness 不同。填到 `references/typescript.md`
> 的深度。结构与那份文件 1:1 对应。

## 1. 栈与配置
- **解析器：** `clap` v4，**derive** API（`#[derive(Parser)]`）。`clap` 是标准。
- 值得拉的辅助：`anyhow`/`thiserror`（错误）、`serde` + `serde_json`（`--json` 信封）、
  `assert_cmd` + `predicates`（e2e 测试）。
- 编译出 → 零启动税；对 agent-per-call spawn 正合适。

## 2. 目录结构
```
src/
  main.rs            # 薄：解析 Cli、dispatch、error → 退出码映射
  cli.rs             # #[derive(Parser)] Cli + Subcommands enum
  output.rs          # Report + emit（text | json | addr）
  error.rs           # CliError { exit_code, next_steps }
  domain/            # 纯核心：selector.rs、ops/（registry + handlers）
  usecases/          # commands/<verb>.rs（薄）、lib.rs（resolve_targets、runner）
```

## 3–10. Harness —— TODO
- **CliError** → `struct CliError { exit_code: i32, next_steps: Vec<String>, .. }` impl
  `std::error::Error`；`main` 返回 `ExitCode`（映射 error）。或用 `thiserror` enum。
- **Report + emit** → `#[derive(Serialize)] struct Report { rows: Vec<Row>, .. }`；`--json` /
  `--format addr` 走同一个 `emit` 函数（addr 路径仅在 `rows` 非空时触发 —— §4 / 坑）。
- **dispatch** → `match cli.command { Cmd::List(a) => command_list(a), .. }`。clap 的 derive 给你
  typed args，于是「归一化 opts」这步基本消失 —— 一个不错的红利。
- **Option mixins** → 一个 `#[derive(Args)] struct Selection { in_, type_, owner, grep }`，用
  `#[command(flatten)]` 铺进读和写两类 subcommand —— 即 §6 的想法，用类型实现。
- **selector + resolve_targets** → 纯 `domain::selector`；`resolve_targets(&doc, &sel)` 共享。
- **op 引擎** → `HashMap<&str, fn(&mut Ctx)>` 或 trait-object registry（§9）。在 `Ctx` 里传一个
  re-entry 闭包来跑子 op，不引入模块环。
- **verb-script** → quote-aware tokenizer（逐 token 记 `quoted: bool`，坑 #1）；共享 template
  builder（坑 #4）。

## 11. 测试
- `assert_cmd`：`Command::cargo_bin("mytool")?.args([...]).assert().code(0)` —— 驱真二进制，正是
  §11 的理念。`do -` 的 stdin 用 `.write_stdin(...)` 喂。

## 12. 环检测
- Rust 的 module/crate 体系天生难成环；让 `domain::ops::*` 别依赖 driver —— 经 `Ctx` 注入 re-entry。
