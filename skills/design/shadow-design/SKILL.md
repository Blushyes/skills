---
name: shadow-design
description: 设计、撰写、评审或改进 CSS 阴影（box-shadow / drop-shadow）与 elevation 体系时使用。触发场景包括："写一下 box-shadow / 这个卡片加个阴影 / 阴影太硬 / 阴影看起来很廉价 / 让这个 modal 浮起来 / 设计 elevation token / hover 浮起效果 / 深色模式下阴影看不见 / shadow looks flat / harsh / cheap / design a shadow system / shadow token / depth / elevation"。本 skill 提供四条设计铁律、多层阴影配方库（sharp/diffuse/dreamy/floating）、完整的 elevation token 设计、组件场景对照表、dark mode 处理方案与常见反模式清单。
---

# Shadow Design — CSS 阴影设计指南

> 综合 Josh Comeau《Designing Beautiful Shadows in CSS》、Tobias Ahlin《Layered Smooth Box Shadows》、Design Systems Surf《Depth With Purpose》以及 Material 3 / Atlassian Design Tokens 的实践提炼。所有配方为合成产物，使用前请按本文档的「调参流程」校准到具体项目。

## 何时使用本 skill

- 用户要求新增/修改 box-shadow、drop-shadow、阴影、卡片浮起、modal 阴影
- 用户描述阴影"太硬、太死、廉价、刀切感、糊一坨、看不出层级"
- 用户设计 design system 的 elevation / depth token
- 用户做 hover 提升、按下凹陷、focus ring 等交互态
- 用户在 dark mode 下处理层级表达
- 用户做组件库（Card / Modal / Dropdown / Popover / Tooltip / Toast）

不要用本 skill 的场景：
- 用户只需要 outline / border / ring（非阴影问题）
- 用户处理 text-shadow（本 skill 主要针对 box / drop shadow）
- 用户要绘制装饰性 SVG 阴影（请用其他设计 skill）

---

## 一、四条设计铁律

### 1. 全局唯一光源
- 全产品约定单一光源方向：通常是「上方偏左、远距离」。
- 因为光源远，所有元素的阴影**角度一致**。
- offset 的水平:垂直比例**全站统一**。常用比例：`0:1`（正下方）或 `1:2`（轻微偏右下）。
- ❌ 反例：A 卡片用 `4px 8px`，B 卡片用 `-2px 6px`，C 卡片用 `0 -4px` —— 用户会觉得画面"飘"。

### 2. 阴影是层级的语言，不是装饰
- 阴影量级 ↔ z 轴高度：离用户越近，**offset 更大、blur 更大、per-layer opacity 更小**。
- 决定用不用阴影的不是「好不好看」，而是「这个元素是否需要表达浮起感」。
- 让 elevation **token 化**：组件根据其语义角色拿对应 token，不要在组件里写一次性的 `box-shadow`。

### 3. 永远用多层阴影
- 单层 box-shadow 即使加大 blur，边缘仍有"刀切感"，本质是一个被高斯模糊的矩形。
- 用 3–6 层叠加，offset 与 blur 通常以 **2 的幂**递增（1, 2, 4, 8, 16, 32），模拟现实光线的散射衰减。
- 层数与 elevation 高度相关：
  - 浅层级（按钮、tag）→ 1–2 层
  - 中层级（卡片、dropdown）→ 3–4 层
  - 高层级（modal、popover）→ 5–6 层

### 4. 阴影颜色匹配上下文
- **黑色阴影是默认错误**：纯黑半透明会把背景脱色，让画面发灰。
- 用 HSL 让阴影"继承"上下文色相：
  - **Hue**：匹配背景或品牌主色
  - **Saturation**：~50–70%（不要 100%，否则像荧光）
  - **Lightness**：~40–55%（深一些但不是黑）
- 实操：用 CSS 变量持有色相分量，每个区域可覆盖：
  ```css
  :root        { --shadow-color: 220 15% 25%; }   /* 中性冷灰，最通用 */
  .blue-card   { --shadow-color: 220 60% 50%; }
  .brand-hero  { --shadow-color: 340 70% 45%; }
  ```

---

## 二、阴影"性格"五型 — 配方库

每个性格都是一种 opacity 曲线 + 一种 offset/blur 关系的组合。geometry 都用 2 的幂作为基准。

### A. Uniform（均匀） — 通用、安全
所有层 opacity 相同，offset 与 blur 翻倍。**作为默认起点**。
```css
.shadow-uniform {
  box-shadow:
    0 1px  1px  hsl(var(--shadow-color) / 0.08),
    0 2px  2px  hsl(var(--shadow-color) / 0.08),
    0 4px  4px  hsl(var(--shadow-color) / 0.08),
    0 8px  8px  hsl(var(--shadow-color) / 0.08),
    0 16px 16px hsl(var(--shadow-color) / 0.08);
}
```

### B. Sharp（锐利） — 近边明确、远处快速消失
opacity 从近到远递减。适合**需要明确边界**的元素（按钮、紧凑卡片）。
```css
.shadow-sharp {
  box-shadow:
    0 1px  1px  hsl(var(--shadow-color) / 0.22),
    0 2px  2px  hsl(var(--shadow-color) / 0.16),
    0 4px  4px  hsl(var(--shadow-color) / 0.10),
    0 8px  8px  hsl(var(--shadow-color) / 0.06),
    0 16px 16px hsl(var(--shadow-color) / 0.03);
}
```

### C. Diffuse（弥散） — 近处虚、远处有外发光
opacity 反向递增，营造柔和外晕。适合**夜间、霓虹、glow 风格**。
```css
.shadow-diffuse {
  box-shadow:
    0 1px  2px  hsl(var(--shadow-color) / 0.06),
    0 2px  4px  hsl(var(--shadow-color) / 0.10),
    0 4px  8px  hsl(var(--shadow-color) / 0.14),
    0 8px  16px hsl(var(--shadow-color) / 0.18);
}
```

### D. Dreamy（梦幻） — blur 远大于 offset
blur ≈ 2× offset，6 层 uniform 低透明。适合 **hero、landing page、需要"漂浮气场"**的大块面。
```css
.shadow-dreamy {
  box-shadow:
    0 1px  2px  hsl(var(--shadow-color) / 0.07),
    0 2px  4px  hsl(var(--shadow-color) / 0.07),
    0 4px  8px  hsl(var(--shadow-color) / 0.07),
    0 8px  16px hsl(var(--shadow-color) / 0.07),
    0 16px 32px hsl(var(--shadow-color) / 0.07),
    0 32px 64px hsl(var(--shadow-color) / 0.07);
}
```

### E. Floating（漂浮） — offset 远大于 blur
垂直距离感强，元素像悬在空中。适合 **modal、drawer、被拖拽的元素**。
```css
.shadow-floating {
  box-shadow:
    0 2px  1px  hsl(var(--shadow-color) / 0.09),
    0 4px  2px  hsl(var(--shadow-color) / 0.09),
    0 8px  4px  hsl(var(--shadow-color) / 0.09),
    0 16px 8px  hsl(var(--shadow-color) / 0.09),
    0 32px 16px hsl(var(--shadow-color) / 0.09);
}
```

**选型口诀**：
- 卡片 → Uniform
- 按钮 → Sharp
- 玻璃/玄学/暗色 → Diffuse
- 大块视觉 → Dreamy
- 浮窗类 → Floating

---

## 三、Elevation Token 系统（可直接落地）

5 级 + 1 个 inset，覆盖 95% 的 UI 需求。geometry 固定，颜色通过 `--shadow-color` 注入。

```css
:root {
  /* 阴影色相分量。每个 surface/section 可覆盖。*/
  --shadow-color: 220 15% 25%;

  /* === Elevation tokens === */
  --elevation-0: none;

  --elevation-1:
    0 1px 1px hsl(var(--shadow-color) / 0.10),
    0 2px 2px hsl(var(--shadow-color) / 0.06);

  --elevation-2:
    0 1px 1px hsl(var(--shadow-color) / 0.08),
    0 2px 2px hsl(var(--shadow-color) / 0.08),
    0 4px 4px hsl(var(--shadow-color) / 0.08);

  --elevation-3:
    0 1px 2px  hsl(var(--shadow-color) / 0.07),
    0 2px 4px  hsl(var(--shadow-color) / 0.07),
    0 4px 8px  hsl(var(--shadow-color) / 0.07),
    0 8px 16px hsl(var(--shadow-color) / 0.07);

  --elevation-4:
    0 1px 2px   hsl(var(--shadow-color) / 0.06),
    0 2px 4px   hsl(var(--shadow-color) / 0.06),
    0 4px 8px   hsl(var(--shadow-color) / 0.06),
    0 8px 16px  hsl(var(--shadow-color) / 0.06),
    0 16px 32px hsl(var(--shadow-color) / 0.06);

  --elevation-5:
    0 2px 4px    hsl(var(--shadow-color) / 0.05),
    0 4px 8px    hsl(var(--shadow-color) / 0.05),
    0 8px 16px   hsl(var(--shadow-color) / 0.05),
    0 16px 32px  hsl(var(--shadow-color) / 0.05),
    0 32px 64px  hsl(var(--shadow-color) / 0.05),
    0 64px 128px hsl(var(--shadow-color) / 0.05);

  /* 内陷（按下、输入框聚焦、凹槽） */
  --elevation-inset:
    inset 0 1px 2px hsl(var(--shadow-color) / 0.12),
    inset 0 2px 4px hsl(var(--shadow-color) / 0.06);
}

/* 语义别名（推荐组件用语义名而非数字） */
:root {
  --shadow-button:   var(--elevation-1);
  --shadow-card:     var(--elevation-2);
  --shadow-card-hover: var(--elevation-3);
  --shadow-dropdown: var(--elevation-3);
  --shadow-popover:  var(--elevation-4);
  --shadow-modal:    var(--elevation-5);
  --shadow-pressed:  var(--elevation-inset);
}
```

**Tailwind v4 用法**（如果用 Tailwind）：
```css
@theme {
  --shadow-card:     /* 同上 */;
  --shadow-modal:    /* 同上 */;
}
/* 使用：<div class="shadow-card hover:shadow-card-hover"> */
```

---

## 四、组件 × Elevation 对照表

| 组件 | Resting | Hover | Active | 备注 |
|---|---|---|---|---|
| Body / page bg | 0 | – | – | 永远平面 |
| Section bg | 0 | – | – | 用 border 或 bg 区分 |
| Inline button | 0 / 1 | 1 / 2 | inset | active 用 inset 模拟按下 |
| Primary button | 1 | 2 | inset | 重点按钮可 +1 级 |
| Tag / Chip | 0 / 1 | 1 | – | 通常无 hover 阴影 |
| Card (static) | 1 | – | – | 单层即可 |
| Card (interactive) | 2 | 3 | 2 | hover ↑1，配 `translateY(-2px)` |
| Dropdown / Menu | 3 | – | – | 比触发器明显高 |
| Popover / Tooltip | 4 | – | – | 用 `filter: drop-shadow` 跟随箭头 |
| Toast / Snackbar | 4 | – | – | 短时浮现 |
| Modal / Dialog | 5 | – | – | 配合 backdrop blur 更强浮起 |
| Drawer / Sheet | 5 | – | – | 侧边滑出 |

---

## 五、交互态规则

```css
.card {
  box-shadow: var(--shadow-card);
  transition: box-shadow 200ms ease, transform 200ms ease;
}
.card:hover {
  box-shadow: var(--shadow-card-hover);
  transform: translateY(-2px);
}
.card:active {
  box-shadow: var(--elevation-1);  /* 按下时降一级 */
  transform: translateY(0);
}
.card:focus-visible {
  /* focus 用 outline，不要改阴影量级 */
  outline: 2px solid hsl(var(--shadow-color) / 0.6);
  outline-offset: 2px;
}
```

**性能警告**：5+ 层的 elevation 做 transition 会触发整块重绘。两个解法：
1. 只 transition `transform`，阴影瞬切；
2. 用「两个叠加层」交叉淡入淡出（resting 与 hover 各放在不同伪元素，过渡 opacity）。

---

## 六、Dark Mode 处理

黑色阴影在深色背景上**几乎不可见**。三种解法：

### 方案 1：surface tonal —— Material 3 推荐
不用阴影，用底色加亮表达提升。Elevation 越高，底色越接近表面色。
```css
[data-theme="dark"] {
  --surface-0: hsl(220 10% 8%);
  --surface-1: hsl(220 10% 12%);  /* +4% lightness */
  --surface-2: hsl(220 10% 15%);
  --surface-3: hsl(220 10% 18%);
  --surface-4: hsl(220 10% 22%);
  --surface-5: hsl(220 10% 26%);
}
.card { background: var(--surface-2); }
.modal { background: var(--surface-5); }
```

### 方案 2：subtle border-glow
高光描边模拟"光线从上方打到边缘"。
```css
[data-theme="dark"] .elevated {
  box-shadow:
    inset 0 1px 0 hsl(0 0% 100% / 0.06),  /* 顶部 1px 亮线 */
    var(--elevation-3);                    /* 同时保留弱阴影 */
}
```

### 方案 3：保留阴影，但用品牌色 + 极低 alpha
```css
[data-theme="dark"] {
  --shadow-color: 220 80% 60%;  /* 用品牌亮色作为阴影色 */
}
/* 各 elevation 的 alpha 全部砍半 */
```

实务建议：**优先用方案 1（surface tonal）**，必要时叠加方案 2 的顶部高光。方案 3 仅在保留品牌氛围时使用。

---

## 七、box-shadow vs filter: drop-shadow

| 场景 | 用 box-shadow | 用 filter: drop-shadow |
|---|---|---|
| 矩形元素（卡片、按钮） | ✅ 默认 | ❌ 性能浪费 |
| 圆角矩形 | ✅ | ⚠️ 也可以 |
| 透明 PNG / SVG | ❌ 会画在矩形包围盒上 | ✅ 沿透明轮廓 |
| 有箭头的 tooltip / popover | ❌ | ✅ 跟随箭头形状 |
| 复杂 clip-path 形状 | ❌ | ✅ |
| 多层叠加 | ✅ 一行多个 | ✅ 多个 drop-shadow 串联 |

```css
/* tooltip 带尖角，必须 drop-shadow */
.tooltip {
  filter:
    drop-shadow(0 1px 2px hsl(var(--shadow-color) / 0.10))
    drop-shadow(0 2px 4px hsl(var(--shadow-color) / 0.08))
    drop-shadow(0 4px 8px hsl(var(--shadow-color) / 0.06));
}
```

**性能坑**：Safari 在 filter 内含 input/textarea 时会引发输入卡顿，避免在表单容器上加 filter。

---

## 八、反模式清单（评审时检查）

- ❌ **黑色单层**：`box-shadow: 5px 5px 10px black`。修：换 HSL、改多层。
- ❌ **光源方向不一致**：负 y offset 与正 y offset 同时出现。
- ❌ **每个组件一次性写阴影**：没走 token。修：抽 elevation 变量。
- ❌ **用阴影代替 z-index**：阴影只表达视觉层级，stacking 用 z-index。
- ❌ **dark mode 直接复用 light 阴影**：黑底加更黑 = 看不见。
- ❌ **阴影是唯一的层级信号**：低视力用户无法感知。务必配合 spacing / border / 背景对比。
- ❌ **5+ 层阴影做 transition**：性能炸。
- ❌ **同一页面 5 种以上 elevation 量级**：决策疲劳，整体显得乱。
- ❌ **阴影 spread > 0 用于柔化**：spread 是扩散硬阴影用的，柔化用 blur。
- ❌ **drop-shadow 套在含 input 的容器上**（Safari 输入延迟）。

---

## 九、调参流程（落地一个项目时）

1. **定光源**：约定 offset 比例（推荐 `0:1` 或 `1:2`），写进 design token 文档。
2. **定基色**：选 1 个中性 `--shadow-color`（如 `220 15% 25%`）作为默认。
3. **定 5 级 elevation geometry**：复制第三节模板。
4. **打开 dark mode 同步设计**：选 surface tonal 或 border-glow 方案。
5. **建语义别名**：`--shadow-card / --shadow-modal` 等。
6. **批量替换组件库**：grep 现有 `box-shadow:`，逐个换 token。
7. **视觉走查**：把所有组件并排显示一屏（Storybook 或专用 page），检查阴影协调性。
8. **a11y 检查**：把阴影全部关掉，UI 是否还能表达层级？不能则补 border / spacing。

---

## 十、参考资料

- Josh Comeau · *Designing Beautiful Shadows in CSS* — 多层叠加 + HSL 色相策略
- Tobias Ahlin · *Layered Smooth Box Shadows* — 几何级数多层配方与 sharp/diffuse 风格
- Design Systems Surf · *Depth With Purpose: How Elevation Adds Realism and Hierarchy* — Elevation 在设计系统中的语义角色
- CSS Script / Shadow.css presets — 8 种 preset 命名启发（uniform / sharp / diffuse / dreamy / shorter / longer）
- Material Design 3 · Elevation & Surface Tint — dark mode surface tonal 方案
- Atlassian Design Tokens · Elevation tokens — 语义化命名实践
