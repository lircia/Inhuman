# 必中大转盘（Wheel）逻辑重设计 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在保持 `Wheel/Wheel.html` 现有外观（白底、双按钮、弹窗、双 canvas、指针样式）不变的前提下，按设计案重建抽奖状态机：两种修改器随机择一；图层分离时**转盘在转、扇区文字不跟盘转**；重力修改器为**整盘同转 + 物理单摆式落点**；每次开抽前静止态必为「指针正上 = 必中项」；抽后 overlay 生动。

**Architecture:** 单一 `Wheel.html` 内用 `wheelCanvas`（色块）与 `textCanvas`（文字）两图层；`drawWheel` 拆为「只画背景」「只画文字」与组合入口，文字是否施加 `currentRot` 由修改器决定。图层分离动画帧**仅更新背景 canvas**，文字层在每帧用**固定角网格**重绘（不乘 `currentRot`），停稳后仍保持 decoupled，**不**在最后一帧强行 `rotate(currentRot)` 贴回扇区（避免「顶格文案闪变」）。重力修改器对**角位移**用阻尼 + 重力矩驱动，平衡位置为必中项中心对准 `-π/2`（指针正上）。随机修改器在 `startSpin` 入口选择。

**Tech Stack:** 纯 HTML/CSS/Canvas 2D API，无构建链、无 npm。

---

## 文件结构（本计划范围内）

| 路径 | 职责 |
|------|------|
| `Wheel/Wheel.html` | 全部 UI、样式、转盘绘制、两种修改器、`rotation` 状态、overlay |
| `.cursor/rules/wheel-lucky-spin-design.mdc` | 约束：不得移除「图层分离」下文字不跟盘转的行为（已由本计划另存） |

---

### Task 1: 绘制 API — 显式「文字是否跟盘」

**Files:**
- Modify: `Wheel/Wheel.html`（`<script>` 内绘制相关函数）

- [ ] **Step 1: 定义绘制入口签名**

在 `drawWheelBackground(currentRot)` 保留现状（仅色块 + `rotate(currentRot)`）。

将 `drawWheelText` 改为显式第二参数，例如：

```javascript
/**
 * @param {number} currentRot 背景层当前角（弧度）；仅当 textFollowsWheel 为 true 时施加到文字层
 * @param {boolean} textFollowsWheel true：与真转盘一致（整盘同转）；false：图层分离（文字角固定在屏幕，不乘 currentRot）
 */
function drawWheelText(currentRot, textFollowsWheel) {
    const n = items.length;
    const arc = (Math.PI * 2) / n;
    textCtx.clearRect(0, 0, 500, 500);
    textCtx.save();
    textCtx.translate(250, 250);
    if (textFollowsWheel) textCtx.rotate(currentRot);
    textCtx.font = 'bold 18px Arial';
    textCtx.textAlign = 'right';
    textCtx.lineJoin = 'round';
    textCtx.miterLimit = 2;
    for (let i = 0; i < n; i++) {
        textCtx.save();
        textCtx.rotate(i * arc + arc / 2);
        textCtx.lineWidth = 4;
        textCtx.strokeStyle = 'rgba(0, 0, 0, 0.35)';
        textCtx.strokeText(items[i], 210, 10);
        textCtx.fillStyle = '#ffffff';
        textCtx.fillText(items[i], 210, 10);
        textCtx.restore();
    }
    textCtx.restore();
}
```

- [ ] **Step 2: 组合函数**

```javascript
function drawWheel(currentRot, textFollowsWheel = true) {
    const n = items.length;
    if (n === 0) {
        wheelCtx.clearRect(0, 0, 500, 500);
        textCtx.clearRect(0, 0, 500, 500);
        return;
    }
    drawWheelBackground(currentRot);
    drawWheelText(currentRot, textFollowsWheel);
}
```

- [ ] **Step 3: 手动验证**

在浏览器打开 `Wheel/Wheel.html`，控制台无报错；改内定项后 `resetToTarget` 仍调用 `drawWheel(rotation, true)`，指针上为必中项（色块 + 字一致）。

---

### Task 2: 静止态与「开抽前」— 必中项对准正上

**Files:**
- Modify: `Wheel/Wheel.html` — `resetToTarget`、`startSpin` 开头

- [ ] **Step 1: 统一目标角公式**

保持（或显式注释）：

```javascript
// 必中项扇区中心对准指针方向（画布向上 = -π/2）
rotation = -(targetIndex * arc + arc / 2) - Math.PI / 2;
```

`resetToTarget` 内：`drawWheel(rotation, true)`。

- [ ] **Step 2: 每次 `startSpin` 在随机修改器前强制对齐**

在 `isSpinning = true` 之后、分支修改器之前插入：

```javascript
resetToTarget(); // 或等价：重算 rotation 并 drawWheel(rotation, true)，保证开抽前必中在正上且整盘可读
```

若 `resetToTarget` 会改 `rotation`，注意与「从当前角缓入动画」的衔接：图层模式可从该 `rotation` 作为 `rotationStart`；重力模式初始角速度可与当前 `rotation` 兼容。

- [ ] **Step 3: 手动验证**

内定「特等奖」→ 静止时 12 点扇区为特等奖且文案一致；点「开始抽奖」前瞬间仍一致。

---

### Task 3: 修改器 A — 图层分离（文字不跟盘转）

**Files:**
- Modify: `Wheel/Wheel.html` — `runLayerModifier`

- [ ] **Step 1: 动画循环内全程 `textFollowsWheel: false`**

```javascript
function runLayerModifier() {
    let startTime = null;
    const duration = 4000;
    const totalSpins = 10 * Math.PI * 2;
    const rotationStart = rotation;

    function animate(time) {
        if (!startTime) startTime = time;
        const progress = Math.min((time - startTime) / duration, 1);
        const easeOut = 1 - Math.pow(1 - progress, 3);
        const currentPos = rotationStart + easeOut * totalSpins;
        drawWheel(currentPos, false);
        if (progress < 1) requestAnimationFrame(animate);
        else {
            rotation = rotationStart + totalSpins;
            drawWheel(rotation, false);
            finishSpin();
        }
    }
    requestAnimationFrame(animate);
}
```

- [ ] **Step 2: 语义注释（防回归）**

在 `runLayerModifier` 上方加注释：**停稳后仍使用 `false`，禁止为「对齐」在最后一帧改为 `true`，否则会破坏设计并造成顶格文案突变。**

- [ ] **Step 3: 手动验证**

连续 3 次抽到图层模式：转时色块动、字相对屏幕角不变；停稳后无整盘文字「甩回」扇区的闪变；弹窗为内定项。

---

### Task 4: 修改器 B — 重力加速（整盘同转 + 物理单摆）

**Files:**
- Modify: `Wheel/Wheel.html` — `runGravityModifier`

- [ ] **Step 1: 状态变量**

在动画闭包内使用（与现有类似，可微调命名）：

```javascript
let vel = 0.45 + Math.random() * 0.25; // 初角速度（rad/frame 量级，可按帧 dt 换算）
const friction = 0.985;   // 角阻尼（每帧乘因子）
const gravity = 0.002;    // 重力矩强度（调手感）
const targetTheta = -(targetIndex * arc + arc / 2) - Math.PI / 2;
```

- [ ] **Step 2: 每帧积分（有阻尼单摆）**

```javascript
function animate() {
    let delta = rotation - targetTheta;
    while (delta > Math.PI) delta -= 2 * Math.PI;
    while (delta < -Math.PI) delta += 2 * Math.PI;
    const gravityEffect = -gravity * Math.sin(delta);
    vel += gravityEffect;
    vel *= friction;
    rotation += vel;
    drawWheel(rotation, true);
    if (Math.abs(vel) < 0.0015 && Math.abs(delta) < 0.02) {
        rotation = targetTheta;
        drawWheel(rotation, true);
        finishSpin();
    } else requestAnimationFrame(animate);
}
```

说明：`sin(delta)` 等效于「必中对侧挂重」的恢复力矩，平衡在 `delta = 0 (mod 2π)`，即必中项在正上。

- [ ] **Step 3: 手动验证**

重力模式多次：可见减速、过冲与回摆；停稳后 12 点为必中项；全程字与盘同角。

---

### Task 5: 修改器选择与 `startSpin` 编排

**Files:**
- Modify: `Wheel/Wheel.html` — `startSpin`

- [ ] **Step 1: 保持随机二选一**

```javascript
const mode = Math.random() > 0.5 ? 'layer' : 'gravity';
if (mode === 'layer') runLayerModifier();
else runGravityModifier();
```

- [ ] **Step 2: 防重入**

保留 `if (isSpinning) return;`；`finishSpin` 内 `isSpinning = false`。

- [ ] **Step 3: 手动验证**

统计约 20 次点击，两种模式均出现；无并发叠帧。

---

### Task 6: Overlay — 生动（外观 DOM 不变，可增强动画）

**Files:**
- Modify: `Wheel/Wheel.html` — `#overlay` 的 CSS 与（可选）`finishSpin` 内联 class

- [ ] **Step 1: CSS 增强（示例，可原样粘贴）**

```css
#overlay { animation: fadeIn 0.35s ease-out; }
#overlay h1 { animation: prizePop 0.55s cubic-bezier(0.34, 1.56, 0.64, 1); }
@keyframes prizePop {
    from { transform: scale(0.6); opacity: 0; filter: blur(6px); }
    to { transform: scale(1); opacity: 1; filter: blur(0); }
}
```

- [ ] **Step 2: `finishSpin` 触发重播（可选）**

```javascript
overlay.style.display = 'flex';
void prizeNameDisplay.offsetWidth; // 强制重排以重触 CSS 动画
prizeNameDisplay.classList.remove('replay');
void prizeNameDisplay.offsetWidth;
prizeNameDisplay.classList.add('replay');
```

若不用 class 技巧，可仅依赖 CSS `animation` 每次 `display:flex` 已足够。

- [ ] **Step 3: 手动验证**

弹窗有淡入 + 奖项名放大/清晰化；点击关闭正常。

---

### Task 7: 边界与 UI

**Files:**
- Modify: `Wheel/Wheel.html`

- [ ] **Step 1: `items.length === 0`**

`drawWheel` 已双清；`startSpin` 早退；`targetSelect` 为空时不崩。

- [ ] **Step 2: 配置 Dialog**

保持现有：逗号分隔项、内定下拉、主界面「设置」「开始抽奖」。

- [ ] **Step 3: 手动验证**

清空奖项再恢复；Esc 关闭设置。

---

### Task 8: 回归清单（无自动化测试时的验收）

**Run:** 浏览器直接打开 `file:///.../Wheel/Wheel.html`（或本地静态服）。

| 步骤 | 操作 | 预期 |
|------|------|------|
| 1 | 内定「特等奖」，静止 | 12 点扇区+字为特等奖 |
| 2 | 开始抽奖直到图层模式 | 色块转、字不跟盘转；停后无字贴回闪变 |
| 3 | 开始抽奖直到重力模式 | 字盘同转；有摆荡感；停于必中 |
| 4 | 弹窗 | 文案为内定项；动画可见 |
| 5 | 改内定为「三等奖」 | 静止 12 点为三等奖 |

---

### Task 9: Git 提交（若仓库需留档）

**Files:** —

- [ ] **Step 1: 查看 diff**

```bash
git status
git diff Wheel/Wheel.html
```

- [ ] **Step 2: 仅暂存本功能相关文件**

```bash
git add Wheel/Wheel.html .cursor/rules/wheel-lucky-spin-design.mdc docs/superpowers/plans/2026-05-15-wheel-rigged-spin-redesign.md
```

- [ ] **Step 3: 提交**

```bash
git commit -m "$(cat <<'EOF'
fix(wheel): restore layer-split spin and document rigged-wheel design

EOF
)"
```

（若用户未要求提交，可跳过 Task 9。）

---

## Self-Review

**1. Spec coverage**

| 设计案条款 | 对应 Task |
|------------|-----------|
| UI 自定义项 + 必中项 | Task 7（已有 UI）、Task 2 |
| 生动 overlay | Task 6 |
| 与真转盘别无二致（静止/重力时） | Task 1–2、Task 4（`true`） |
| 开抽前指针上为必中 | Task 2 |
| 指针正上、抽奖时盘转 | 现有 DOM/CSS + Task 3–4 |
| 图层分离修改器 | Task 1、3 |
| 重力修改器 + 物理 | Task 4 |
| 随机修改器 | Task 5 |
| 不删图层分离逻辑（团队约束） | `.mdc` + Task 3 注释 |

**2. Placeholder scan** — 无 TBD；物理参数为可调初值并注明调参。

**3. Type一致性** — `textFollowsWheel` 与 `drawWheel` 第二参数全文一致。

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-05-15-wheel-rigged-spin-redesign.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — 每任务派生子代理、任务间复核、迭代快  

**2. Inline Execution** — 本会话按 `executing-plans` 顺序执行并设检查点  

**Which approach?**

若选 **Subagent-Driven**：请使用 **superpowers:subagent-driven-development**。  
若选 **Inline Execution**：请使用 **superpowers:executing-plans**。
