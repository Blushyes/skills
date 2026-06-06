# TypeScript CLI 参考（commander）

把 `SKILL.md` 里的原则落成具体技术栈 + 可直接复制的构件。抄过去，别重造。从一个生产级
agent-facing CLI 泛化而来。

## 目录
1. [栈与配置](#1-stack--setup)
2. [目录结构](#2-directory-layout)
3. [错误类型 —— `CliError`](#3-the-error-type--clierror)
4. [输出信封 —— `Report` + `emit`（text / json / addr）](#4-the-output-envelope)
5. [dispatch harness](#5-the-dispatch-harness)
6. [Option mixins（`withCommonOpts`、`withSelection`）](#6-option-mixins)
7. [selector：语法、解析器、共享 resolver](#7-the-selector)
8. [selector-edit runner（dry-run / --apply / blast radius）](#8-the-selector-edit-runner)
9. [op 引擎：dispatch map（不是 1000 行 if/else）](#9-the-op-engine-dispatch-map)
10. [verb-script：tokenizer（quote-aware）→ ops](#10-the-verb-script)
11. [测试：spawn 编译出的二进制](#11-testing)
12. [环检测](#12-cycle-guard)

---

## 1. Stack & setup

- **运行时：** Node ≥20，**ESM**（`"type": "module"`）。
- **模块：** `NodeNext` —— 即使源码是 `.ts`，本地 import 也带 **`.js` 后缀**
  （`import { x } from "./foo.js"`）。每个人都会被绊一次。
- **构建：** 纯 `tsc`（无 bundler）。新文件自动编译；`dist/` 镜像 `src/`。bundler 对 CLI 没好处、
  还把 source map 搞复杂。
- **解析器：** `commander`（v12+）。
- **测试：** `vitest`，把 *编译出的* `dist/bin.js` 作为子进程 spawn（§11）。

`package.json`（要点）：
```jsonc
{
  "type": "module",
  "bin": { "mytool": "./dist/bin.js" },
  "scripts": { "build": "tsc", "pretest": "tsc", "test": "vitest run", "typecheck": "tsc --noEmit" },
  "dependencies": { "commander": "^12.1.0" },
  "devDependencies": { "typescript": "^5.5.0", "vitest": "^2.0.0", "madge": "^7.0.0", "@types/node": "^20" }
}
```
`tsconfig.json`（要点）：`"module": "NodeNext"`、`"moduleResolution": "NodeNext"`、
`"target": "ES2022"`、`"rootDir": "./src"`、`"outDir": "./dist"`、`"strict": true`、
`"declaration": true`。

`bin.ts` 是个 10 行 shim —— 别往里塞逻辑：
```ts
#!/usr/bin/env node
import { main } from "./cli.js";
main(process.argv.slice(2)).then((code) => process.exit(code))
  .catch((err) => { console.error("RUNTIME FAILED:", err?.message ?? err); process.exit(70); });
```

## 2. Directory layout

```
src/
  bin.ts                 入口 shim（无逻辑）
  cli.ts                 commander 接线 + 全局 flag 解析 + dispatch + emit
  common.ts              CliError、Report 类型、退出码、共享 IO
  output.ts              renderText / emit（text | json | addr）
  domain/                纯、无框架的核心（无 commander、无 fs 副作用）
    selector.ts          选择语法 + 解析器
    address.ts           地址解析
    ops/                 op 引擎（见 §9）
      types.ts  registry.ts  apply.ts  ops-content.ts  ops-entity.ts  …
  usecases/              编排：load → apply → save → report
    session.ts           加载/保存/校验文档
    lib.ts               共享 helper（resolveTargets、runner、verb-script）
    commands/            一动词一文件（薄壳）
      list.ts  replace.ts  set-type.ts  do.ts  patch.ts  …
  infra/                 IO 适配（fs store、远端 store、env）
```
依赖只指向下：`bin → cli → usecases → domain → infra`。用 madge 守（§12）。`cli.ts` 从
`usecases/commands/<verb>.js` deep-import 每个命令。

## 3. The error type — `CliError`

每个失败都是一个携带 **退出码** 和 **下一步** 的 typed error。这就是「错误即下一条命令」
（SKILL 第 1 部分）落地的方式。

```ts
// common.ts
export const EXIT_OK = 0;        // changed / success
export const EXIT_NOMATCH = 1;   // nothing matched, nothing done (grep-style)
export const EXIT_NEEDS_FIX = 2; // applied but validation needs repair
export const EXIT_USAGE = 64;    // bad invocation
export const EXIT_INPUT = 66;    // bad input data
export const EXIT_RUNTIME = 70;  // unexpected

export class CliError extends Error {
  exitCode: number; errorCode?: string;
  required: string[]; received: string[]; nextSteps: string[]; op?: string;
  constructor(title: string, message: string, opts: {
    exitCode?: number; errorCode?: string; required?: string[];
    received?: string[]; nextSteps?: string[]; op?: string;
  } = {}) {
    super(message);
    this.name = title; this.exitCode = opts.exitCode ?? EXIT_USAGE;
    this.errorCode = opts.errorCode; this.required = opts.required ?? [];
    this.received = opts.received ?? []; this.nextSteps = opts.nextSteps ?? [];
    this.op = opts.op;
  }
}
```
用法 —— 注意 `nextSteps` 装着修法：
```ts
throw new CliError("REPLACE BLOCKED: ambiguous match", "matched 3 times in one item.", {
  exitCode: EXIT_USAGE, errorCode: "FROM_AMBIGUOUS",
  received: ["3 matches"],
  nextSteps: ["Pass --all to replace every occurrence, or a longer unique --from."],
});
```

## 4. The output envelope

一个 `Report` 形状，三种渲染。`rows[]` 数组是 agent 的结构化输入；格式化的 `result[]` 字符串给
人看。**把地址从正文里拿出来**（SKILL 第 1 部分）。

```ts
// common.ts
export interface Report {
  title?: string; op?: string; changed?: boolean; summary?: string;
  result?: string[];                        // human lines
  rows?: Array<Record<string, unknown>>;    // structured: { addr, type, ... } per row
  issues?: string[]; next?: string[];
  [key: string]: unknown;                   // index sig → attach fields freely; JSON passes them through
}
```
```ts
// output.ts
export function emit(report: Report, exitCode: number, jsonMode: boolean, format?: string): void {
  // bare address stream — only when the report actually has rows; otherwise fall
  // through so a write verb's report is never silently swallowed (SKILL Pitfall, §5).
  if (format === "addr" && Array.isArray(report.rows)) {
    const addrs = report.rows.map((r) => String(r.addr ?? "")).filter(Boolean);
    process.stdout.write(addrs.length ? addrs.join("\n") + "\n" : "");
    return;
  }
  if (jsonMode) {
    process.stdout.write(JSON.stringify({ ...report, exit_code: exitCode, ok: exitCode === EXIT_OK }, null, 2) + "\n");
    return;
  }
  process.stdout.write(renderText(report, exitCode));   // terse bulleted text
}
```

## 5. The dispatch harness

把 `commander` 的 action 回调收成统一的 handler 签名：`(opts) => Promise<[Report, number]>`。
`dispatch` 归一化 options、把 positional 收进 `_args`、调 handler、emit。

```ts
// cli.ts
type Handler = (opts: Record<string, unknown>) => Promise<[Report, number]> | [Report, number];

function dispatch(handler: Handler, state: { jsonMode: boolean; format?: string; exitCode: number }) {
  return async (...args: unknown[]) => {
    const cmd = args[args.length - 1] as Command;
    const positionals = args.slice(0, -2).filter((p) => p !== undefined) as string[];
    const opts = camelToSnake(cmd.opts());     // --dry-run → dry_run
    if (positionals.length) opts._args = positionals;
    const [report, code] = await handler(opts);
    emit(report, code, state.jsonMode, state.format);
    state.exitCode = code;
  };
}
```

**手解析全局 flag —— 做对**（SKILL 坑 #5）。扫一遍、支持 `=` 形式、绝不偷走下一个 flag：
```ts
export async function main(argv: string[]): Promise<number> {
  let args = [...argv];
  let format: string | undefined;
  for (let i = 0; i < args.length; i++) {
    const a = args[i]!;
    if (a.startsWith("--format=")) { format = a.slice(9); args.splice(i, 1); i--; }
    else if (a === "--format") {
      const v = args[i + 1];
      if (v !== undefined && !v.startsWith("-")) { format = v; args.splice(i, 2); } // don't eat a flag
      else args.splice(i, 1);
      i--;
    }
  }
  const state = { jsonMode: args.includes("--json"), format, exitCode: EXIT_OK };
  args = args.filter((a) => a !== "--json");
  const program = buildProgram(state);
  program.exitOverride().allowUnknownOption(false).allowExcessArguments(false);
  try { await program.parseAsync(args, { from: "user" }); }
  catch (e) { return emitError(e as CliError, state.jsonMode); }
  return state.exitCode;
}
```

## 6. Option mixins

别在 40 个命令里重复声明 option —— 组合它们。两个大头：通用 IO flag，以及 **读写共享的选择
flag**（SKILL 第 2 部分）。

```ts
// cli.ts
function withCommonOpts(cmd: Command): Command {
  return cmd.option("--json").option("--path <p>").option("--dry-run").option("--apply");
}
// the SAME flags drive `list` (read) and `set-type` (write):
function withSelection(cmd: Command): Command {
  return cmd
    .option("--in <selector>")   // single | a..b | a..b,c..d | '*'
    .option("--type <t>")        // predicate
    .option("--owner <id>")      // predicate
    .option("--grep <re>")       // pattern
    .option("--has <facet>");    // predicate
}
// query AND write verbs both get it:
withSelection(withCommonOpts(program.command("list"))).action(dispatch(commandList, state));
withSelection(withCommonOpts(program.command("set-type [at] [value]"))).action(dispatch(commandSetType, state));
```
注意写动词上的 `[at]` 是 **可选** 的：给了 → 单点；没给 + 选择 flag → sweep。

## 7. The selector

一个 selector 是 region 的逗号列表；每个 region 是一个点或 `lo..hi` 区间；`'*'` 是显式的全本
哨兵。放在 `domain/` —— 纯、可测、无 IO。

```ts
// domain/selector.ts
export interface Selector {
  readonly isEmpty: boolean;
  matches(addr: ParsedAddress): boolean;
}
export const WHOLE = { isEmpty: false, matches: () => true } satisfies Selector; // '*': matches all, but NOT empty
class Regions implements Selector {
  constructor(readonly regions: Region[]) {}
  get isEmpty() { return this.regions.length === 0; }
  matches(a: ParsedAddress) { return this.regions.length === 0 ? true : this.regions.some((r) => inRegion(r, a)); }
}
export function parseSelector(raw: string): Selector {
  const v = raw.trim();
  if (!v) return new Regions([]);          // empty: matches all but isEmpty=true (callers refuse a bare dump)
  if (v === "*") return WHOLE;             // explicit whole-scope
  return new Regions(v.split(",").map((seg) => parseRegion(seg.trim())));
}
```
要紧的设计注解：
- **`isEmpty` vs `WHOLE`。** *空* selector 匹配一切，但 `isEmpty` 为真 → 调用方拒绝它（一次裸的、
  危险的 dump）。`'*'` 匹配一切，但 `isEmpty` 为假 → 一个刻意的全本范围。别把两者混为一谈，并留意
  这个不对称：一个「仅当给了 scope 时才收窄」的函数必须判 `!isEmpty && selector !== WHOLE`，而不只是
  `!isEmpty`（否则就是个真 bug）。
- **区间端点必须同 kind**，并拒绝跨层裸数字（`3..7` → 强制 `ep_3..ep_7`）。reversed 区间 **自动 swap**
  （id 单调；无歧义、比报错对 agent 更友好）。

**共享 resolver** —— 基石。一个函数把一个选择（selector + 谓词）变成命中的地址，*读命令和每个写
动词都用它*。这就是让「读写共享选择」在代码里成真的东西：
```ts
// usecases/lib.ts
export function resolveTargets(doc: Doc, opts: Record<string, unknown>): Target[] {
  const sel = parseSelector(String(opts.in ?? ""));
  const type = str(opts.type), owner = str(opts.owner), grep = buildGrep(str(opts.grep));
  const out: Target[] = [];
  for (const item of doc.items) {
    if (!sel.matches(item.addr)) continue;
    if (type && item.type !== type) continue;        // predicates, AND-combined
    if (owner && item.owner !== owner) continue;
    if (grep && !grep(item.content)) continue;
    out.push(item);
  }
  return out;
}
export function hasAnySelection(opts: Record<string, unknown>): boolean {
  return !parseSelector(String(opts.in ?? "")).isEmpty
    || !!(opts.type || opts.owner || opts.grep || opts.has);
}
```

## 8. The selector-edit runner

共享的写路径：解析 targets，然后对每个套用一个 op template —— 带 dry-run-by-default 护栏
（SKILL 第 4 部分）。一个写动词是个薄壳：造 template、调它。

```ts
// usecases/lib.ts
const SKIPPABLE = new Set(["NO_MATCH_HERE", "NOT_APPLICABLE"]); // per-item "doesn't apply", not fatal

export async function applySelectorEdits(opts, template: Op, verb: string): Promise<[Report, number]> {
  const apply = Boolean(opts.apply);
  const session = await load(opts);
  const targets = resolveTargets(session.doc, opts);
  let applied = 0, skipped = 0;
  for (const t of targets) {
    try { applyOps(session.doc, [{ ...template, at: t.addr }], { renumber: false }); applied++; } // renumber once, not per-item
    catch (e) { if (e instanceof CliError && SKIPPABLE.has(e.errorCode!)) { skipped++; continue; } throw e; }
  }
  if (applied) renumber(session.doc);
  const blast = [`verb: ${verb}`, `matched: ${targets.length}`,
                 `${apply ? "applied" : "would-apply"}: ${applied}`, `skipped: ${skipped}`,
                 `targets: ${targets.slice(0, 12).map((t) => t.addr).join(", ")}`];
  if (!apply) {                                   // DRY-RUN by default
    const v = validate(session.doc);              // validate in-memory would-be state (Pitfall #3)
    return [{ title: "DRY-RUN (no write)", changed: false, result: [...blast, "pass --apply to commit"] },
            v.passed ? EXIT_OK : EXIT_NEEDS_FIX];
  }
  if (!targets.length) return [{ title: "no match", changed: false, result: blast }, EXIT_NOMATCH];
  await save(session, opts);
  const v = validate(session.doc);
  return [{ title: "APPLIED", changed: applied > 0, result: blast }, v.passed ? EXIT_OK : EXIT_NEEDS_FIX];
}
```
一个写命令（薄壳），单点和 sweep 复用同一个 template：
```ts
// usecases/commands/set-type.ts
export async function commandSetType(opts) {
  const args = (opts._args as string[]) ?? [];
  const sweep = hasAnySelection(opts);
  const value = sweep ? args[0] : args[1];        // sweep: `set-type <value>`; point: `set-type <at> <value>`
  if (!value) throw new CliError("set-type BLOCKED", "missing <value>", { errorCode: "ARGS_MISSING" });
  const template = { op: "type.set", type: value };
  if (sweep) return applySelectorEdits(opts, template, "set-type");
  return applySingleOp(opts, { ...template, at: args[0] });
}
```

## 9. The op engine: dispatch map (not a 1000-line if/else)

每个变更 —— 来自 CLI、verb-script、或 JSON —— 都编译到一个 op、走同一个引擎。把引擎实现成
**registry**，不是一个巨型 conditional（SKILL 第 5 部分）。

```ts
// domain/ops/types.ts
export interface OpCtx { doc: Doc; op: Op; applied: string[]; kind: string;
  runOps: (ops: Op[]) => string[]; }     // injected re-entry → ops never import apply.ts (no cycle!)
export type OpHandler = (ctx: OpCtx) => void;
```
```ts
// domain/ops/ops-content.ts  (one file per group; handler bodies are verbatim logic)
export const op_content_replace: OpHandler = ({ doc, op, applied, kind }) => { /* … */ applied.push(kind); };
export const op_type_set: OpHandler = ({ doc, op, applied, kind }) => { /* … */ applied.push(kind); };
```
```ts
// domain/ops/registry.ts
import * as content from "./ops-content.js"; import * as entity from "./ops-entity.js";
export const REGISTRY: Record<string, OpHandler> = {
  "content.replace": content.op_content_replace,
  "type.set": content.op_type_set,
  "entity.merge": entity.op_entity_merge,
  // …one line per op
};
```
```ts
// domain/ops/apply.ts  — the entire driver. The 1000-line dispatcher is gone.
export function applyOps(doc: Doc, ops: Op[], opts: { renumber?: boolean } = {}): string[] {
  const applied: string[] = [];
  for (const op of ops) {
    const kind = String(op.op);
    const handler = REGISTRY[kind];
    if (!handler) throw new CliError("OP BLOCKED", `unsupported op: ${kind}`, { errorCode: "OP_UNSUPPORTED" });
    handler({ doc, op, applied, kind, runOps: (sub) => applyOps(doc, sub) }); // runOps lets an op re-enter for sub-ops
  }
  if (opts.renumber !== false) renumber(doc);  // once at the end; skippable for sweeps (§8)
  return applied;
}
```
为什么 `runOps` 经 context 注入而不是 import：需要调用子 op 的 `ops-*.ts`（如 `entity.delete`
委派给 `entity.merge`）否则会 `import` `apply.ts`，而 apply 又 import `registry.ts`、registry 又 import
`ops-*.ts` → 一个 import 环。把 re-entry 注入进去，环就消失了。

## 10. The verb-script

`do -` / `do file`：一行一条 CLI 动词，用一个 quote-aware 的 tokenizer 解析、编译到 *同一批* op。
这是 agent 友好的批量 surface（SKILL 第 3 部分）。

**追踪引号的 tokenizer**（SKILL 坑 #1 —— 最贵那个）：
```ts
export interface Token { text: string; quoted: boolean }   // quoted = any char came from quotes/escape
export function tokenize(line: string): Token[] {
  const out: Token[] = []; let cur = "", started = false, quoted = false, sq = false, dq = false;
  const flush = () => { if (started) { out.push({ text: cur, quoted }); cur = ""; started = false; quoted = false; } };
  for (let i = 0; i < line.length; i++) {
    const ch = line[i]!;
    if (sq) { if (ch === "'") sq = false; else { cur += ch; quoted = true; } }
    else if (dq) { if (ch === '"') dq = false; else if (ch === "\\" && i + 1 < line.length) { cur += line[++i]; quoted = true; } else { cur += ch; quoted = true; } }
    else if (ch === "'") { sq = true; started = true; }
    else if (ch === '"') { dq = true; started = true; }
    else if (ch === "\\" && i + 1 < line.length) { cur += line[++i]; started = true; quoted = true; }
    else if (ch === " " || ch === "\t" || ch === "\r") flush();
    else { cur += ch; started = true; }
  }
  flush(); return out;
}
const isFlag = (t: Token) => !t.quoted && t.text.startsWith("--");  // a quoted "--x" is a VALUE, not a flag
```
**解析**（按动词的 flag 元数；`=` 形式含布尔 `--flag=false`；缺值守卫；动词不是 flag 守卫 ——
SKILL 坑 #2、#5）：
```ts
export function parseLine(tokens: Token[], boolFlags: Set<string>, repeatFlags: Set<string>) {
  if (!tokens.length || isFlag(tokens[0]!)) throw lineErr("a line must start with a verb");
  const verb = tokens[0]!.text, args: string[] = [], flags: Record<string, unknown> = {};
  for (let i = 1; i < tokens.length; i++) {
    const t = tokens[i]!;
    if (!isFlag(t)) { args.push(t.text); continue; }
    let name = t.text.slice(2); const eq = name.indexOf("=");
    if (eq >= 0) {                                   // --flag=value
      const val = name.slice(eq + 1); name = name.slice(0, eq);
      if (!name) throw lineErr(`empty flag name in "${t.text}"`);
      flags[name] = boolFlags.has(name) ? !["false","0","no","off"].includes(val.toLowerCase()) : val;
      continue;
    }
    if (boolFlags.has(name)) { flags[name] = true; continue; }
    const next = tokens[i + 1];
    if (next === undefined || isFlag(next)) throw lineErr(`--${name} needs a value`); // don't swallow a flag
    flags[name] = tokens[++i]!.text;
  }
  return { verb, args, flags };
}
```
**编译到 op，复用 CLI handler 用的同一批 builder**（SKILL 坑 #4 —— 单一真相源）。多态动词对不支持
的地址 kind *大声* 拒绝（SKILL 坑 #6），而不是路由到错误的 op：
```ts
export function lineToOp(verb: string, args: string[], flags: Record<string, unknown>): Op {
  const a0 = str(args[0]), a1 = str(args[1]);
  switch (verb) {
    case "replace":  return { op: "content.replace", at: a0, from: str(flags.from), to: flags.to ?? "" };
    case "set-type": return { op: "type.set", at: a0, type: a1 };
    // mutex/defaults live in ONE shared builder, called by both this and the CLI handler:
    case "relate":   return { ...relateTemplate(a1, flags), at: a0 };
    case "delete": {
      const k = parseAddress(a0).kind;
      if (k === "item") return { op: "item.delete", at: a0 };
      if (k === "entity") return { op: "entity.delete", target: a0 };
      throw lineErr(`delete cannot operate on a ${k} address (${a0})`); // explicit, not a silent wrong op
    }
    default: throw lineErr(`unknown verb '${verb}'`);
  }
}
```
**作为一个事务运行**（只在被管道喂入时读 stdin —— 别在 TTY 上阻塞）：
```ts
// usecases/commands/do.ts
export async function commandDo(opts) {
  const file = str((opts._args as string[])?.[0]);
  let text: string;
  if (!file || file === "-") {
    if (process.stdin.isTTY) throw new CliError("DO BLOCKED", "no script piped to stdin", { errorCode: "NO_INPUT" });
    text = fs.readFileSync(0, "utf-8");
  } else text = fs.readFileSync(file, "utf-8");
  const ops: Op[] = [];
  text.split("\n").forEach((raw, ln) => {
    const line = raw.trim(); if (!line || line.startsWith("#")) return;
    try { const { verb, args, flags } = parseLine(tokenize(raw), BOOL[verbOf(raw)] ?? S, REPEAT[verbOf(raw)] ?? S);
          ops.push(lineToOp(verb, args, flags)); }
    catch (e) { throw new CliError("DO BLOCKED", `line ${ln + 1}: ${(e as Error).message}`, { errorCode: "DO_LINE_INVALID" }); }
  });
  const session = await load(opts);
  applyOps(session.doc, ops);                       // ONE transaction: any throw → nothing saved
  if (!opts.apply) return [{ title: "DO DRY-RUN", changed: false, result: [`${ops.length} ops would apply`] }, EXIT_OK];
  await save(session, opts);
  return [{ title: "DO APPLIED", changed: ops.length > 0, result: [`applied ${ops.length} ops`] }, EXIT_OK];
}
```

## 11. Testing

把 **编译出的** 二进制作为子进程 spawn —— 这是最真实的测试，能抓到单测漏掉的参数解析/退出码
bug。用 `--path` 驱一份本地文件，就不用起 server。

```ts
// tests/helpers/run.ts
import { execFile } from "node:child_process"; import { promisify } from "node:util";
const exec = promisify(execFile);
export async function run(args: string[], cwd: string) {
  try { const { stdout } = await exec("node", ["dist/bin.js", "--json", ...args], { cwd, encoding: "utf-8" });
        return { stdout, status: 0 }; }
  catch (e: any) { return { stdout: e.stdout ?? "", status: e.code ?? 1 }; }
}
```
```ts
// tests/edits.test.ts
it("sweep defaults to dry-run and writes nothing without --apply", async () => {
  const res = await run(["replace", "--in", "*", "--from", "x", "--to", "y"], tmp);
  expect(res.status).toBe(0);
  expect(JSON.parse(res.stdout).changed).toBe(false);
  expect(readDoc().includes("x")).toBe(true);          // unchanged
});
it("rejects a mutex conflict in a do-script with a line number", async () => {
  fs.writeFileSync(`${tmp}/e.txt`, "relate a:b --to s --clear\n");
  const res = await run(["do", `${tmp}/e.txt`, "--apply"], tmp);
  expect(res.status).toBe(64);
  expect(res.stdout).toContain("line 1");
});
```
stdin（`do -` / `patch -`）用 `execFileSync(..., { input })`。

## 12. Cycle guard

架构唯一的硬规则（无 import 环）守起来很便宜：
```bash
npx madge --circular --extensions js dist   # → "✔ No circular dependency found!"
```
加进 CI。一旦失败，常见元凶是某个 `ops-*.ts` import 了 `apply.ts` —— 改成经 context 注入 `runOps`
（§9）。
