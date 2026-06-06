# Go CLI 参考（cobra）—— 骨架

> 状态：骨架。`SKILL.md` 里的原则原样适用；只有 harness 不同。等真要做 Go CLI 时，把下面各节
> 填到 `references/typescript.md` 的深度。结构与那份文件 1:1 对应，方便互相参照。

## 1. 栈与配置
- **框架：** `spf13/cobra`（kubectl / gh / docker / helm —— 标准）。需要配置就配 `viper`。
- **为什么 cobra：** 成熟、Go 的事实约定、有生成器（`cobra-cli`），且 `RunE`/`Options`-struct 的
  分离 *强制* 你写薄命令壳。
- 编译出的二进制 → **零启动税**，对 agent-per-call 的 spawn 正合适。

## 2. 目录结构（cobra 约定）
```
cmd/                 # main 包，一命令一文件（薄壳）
internal/            # 业务逻辑（"domain"/"usecases"），不导出
  ops/               # op registry/dispatch（对应 TS 参考 §9）
  selector/          # 选择语法
pkg/                 # 对外可复用库（若有）
main.go              # 3 行：cmd.Execute()
```
gh 的实际拆法是黄金标准：命令层在 `pkg/cmd/<noun>/<verb>/`，业务在 `internal/`，依赖经一个
`Factory` + `iostreams` 注入。

## 3–10. Harness —— TODO
把 TS 构件映射成 Go 惯用法：
- **CliError** → 一个自定义 `error` 类型，带 `ExitCode`、`NextSteps []string`；在 `main` 里
  `os.Exit(asExit(err))`。cobra：让 `RunE` 返回 error，退出码映射在 `main` 做。
- **Report + emit** → 一个 `Report` struct；`--json` 用 `encoding/json`；`--format addr` 打印
  `rows[].addr`。输出走注入的 `iostreams`（gh 模式）方便测试。
- **dispatch harness** → cobra 的 `RunE` 就是 harness；保持它「解析 flag → 造 Options → 调
  Options 上的方法 → 格式化」。`RunE` 里不放逻辑。
- **Option mixins** → 一个 `func addSelectionFlags(cmd *cobra.Command)`，读/写命令复用（§6 的想法）；
  全局 `--json/--format` 用 persistent flag。
- **selector + resolveTargets** → 一个纯 `selector` 包；`ResolveTargets(doc, opts)` 被 list 和每个写
  命令共享（基石，§7）。
- **op 引擎** → `map[string]OpHandler` registry + 一个 driver（§9）。经一个 context struct 注入
  re-entry，避免 import 环（Go 直接禁止环 —— 在这里是优点）。
- **verb-script** → 一个 quote-aware tokenizer（逐 token 记 `quoted`，§10，坑 #1）；用共享 template
  builder 编译到同一批 op。

## 11. 测试
- `go test`；e2e 就构建二进制并 exec（`os/exec`），和 TS 参考一样；或用 cobra 的 in-process
  `cmd.SetArgs(...)` + `Execute()`、捕获 `iostreams` 的 buffer。

## 12. 环检测
- Go 在编译期禁止 import 环 —— 免费。让 `internal/ops/*` 别 import driver；注入 re-entry。
