# GOODBUDDY · 公益 Agent 的人格化外壳

一句话：做一件好事，就立刻拥有一只有性格、可分享、可能是 SSR 的小生物。

GOODBUDDY 是一个**独立 skill**，挂在公益 Agent 上做"人格化装饰层"——不污染底层数据卡，只在主 Agent 的文本气泡和卡片 sig 区注入 Buddy 的语气、ASCII 形象、订阅召回。

哲学层面是 **赠送品 ≠ 养成品**：用户做了第一件好事就直接拿到一只 Buddy（含物种 × 性格 × 稀有度），不再需要任何"前置养成"，关系通过持续做好事一起累积。

---

## 文件

| 文件 | 用途 |
|---|---|
| `GOODBUDDY-demo-v6.html` | 主交互 demo（单文件 · 无依赖 · 浏览器打开即可） |
| `47-GOODBUDDY-Skill-MVP-v3.md` | Skill MVP 设计文档（v3） |

## 本地预览

demo 是单文件 HTML，无构建：

```bash
# 推荐：跑个 http server 避开 file:// 协议限制
python3 -m http.server 8765
# 然后浏览器打开 http://localhost:8765/GOODBUDDY-demo-v6.html
```

或者直接双击 `GOODBUDDY-demo-v6.html` 用浏览器打开。

## 当前 demo 包含什么

**Scene 1 · Agent 内主动召唤**
- 输入 `@公益伙伴` / `GoodBuddy` 触发
- 已激活：直接拉起 Buddy 状态半屏
- 未激活：露出 🥚 等待孵化卡（不露形象 · 保留神秘感），先做一件好事换 1 朵 🌸 才能破壳

**Scene 2 · 激活前后对比**
- 同一条查询 "帮我查查我的小红花" 在 buddyMode OFF / ON 下的差异
- OFF：干巴巴数据卡 + 普通签名
- ON：Buddy 用自己人格说话（文本气泡）+ 同一张 1:1 数据卡 + Buddy 主题色 sig
- ON 还能这样：早 8 点 Buddy 主动召回（订阅触发）

**Scene 3 · 分享卡预览**
- 5 只小生物的卡面缩略

**Buddy 状态半屏**
- ASCII 形象（text-shadow 呼吸发光 · 按稀有度染色）
- 名字 / 物种 / 稀有度 / 关系称谓
- 一句话 quote（按性格走 + 偶尔提及最近行为）
- 🌸 进度条 → 下一只召唤
- 今天还能再做（3 行任务 · 完成后 +1🌸 + ✓ 已收下）
- 我们一起做过的（timeline 日记风 · 复用三大 skill 数据 · 不再用 emoji）
- 底部入口：图鉴 / 订阅明日

## 数据模型（demo 用的 mock）

```js
MOCK_USER = {
  has_buddy: true,
  buddy_mode: true,
  flower_balance: 27,
  flower_to_summon: 30,
  interaction_count: 8,
  wechat_nickname: '阿明',
  recent_deeds: [ /* 复用三大 skill 数据 */ ],
  completed_tasks: { step: false, answer: false, donate: false },
  pending_inflow: 0
}
```

Buddy 实体来自 `generate(seed)`：物种 × 性格独立抽取 · 稀有度 N/R/SR/SSR · 名字按物种取。

## 接入接口（设计稿）

```
goodbuddy.getBuddySig({ skill, payload })   → 主 Agent 卡 sig 区注入
goodbuddy.getBuddyStatus()                  → 主动召唤拉半屏
goodbuddy.claimBuddy({ trigger_deed_id })   → 第一件好事破壳
goodbuddy.nameBuddy({ name })               → 用户命名
goodbuddy.summonNext()                      → 攒满 30 朵后召唤下一只
goodbuddy.subscribeRecall({ time })         → 订阅明日召回
```

详见 `47-GOODBUDDY-Skill-MVP-v3.md`。

## 版本

当前：**v6.13** · 见 demo 顶部 lab label。
