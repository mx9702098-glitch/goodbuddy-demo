---
name: GOODBUDDY Skill MVP
version: v3
date: 2026-05-16
status: 待对齐（方老师 / 袁康）
覆盖: 31-GOODBUDDY-产品设计思路-v2 的"养成"倾向部分
对齐目标: 5/25 原子能力就绪，6 月初首版小入口发布
---

# 47-GOODBUDDY-Skill-MVP-v3

## 0. 文档定位与版本差异

本文档是 GOODBUDDY 作为 sub-agent skill 的 MVP 工程骨架，给方老师做产品对齐、给袁康做接口落地。

**与 v2 核心宪法的关键差异**（v2 → v3）：

| 维度 | v2（31 号文件） | v3（本文档） |
|---|---|---|
| 产品哲学 | 蚂蚁森林化 · 认领 → 孵化 → 成长 → 喂养 | 赠送品 · 即时获得完整 Buddy · 关系演进而非养成 |
| 孵化机制 | 10 天孵化 | **彻底砍掉**，即时出生 |
| 货币系统 | 善意值（独立） | 统一走小红花，砍善意值 |
| 留存钩子 | 喂养 → Buddy 长大 | 关系演进（称谓变化 + Buddy 偶尔记得你做过的事）+ 订阅召回 + 集卡 |
| 第一印象优先级 | 中（需投入才理解价值） | **高**（看一眼就该想截图） |

v2 的"真功德原则"、"用户不会主动来"、"接地气文案"、"不引设计师"等基调全部保留。

---

## 1. 产品定位（一句话）

> 用户做了一件好事 → 立刻获得一只完整、有性格、可分享、可能是 SSR 的 ASCII 小生物。Buddy 本身就是产品价值，公益行为是 Buddy 的入场券，也是 Buddy 跟用户关系演进的纽带。

**核心 loop（4 条并行 driver，最终都收敛到"做一件好事"）**：

| 用户类型 | driver | 怎么承接 |
|---|---|---|
| 关系派 | "TA 越来越懂我" | 关系称谓 4 档 + Buddy 偶尔提及近期行为 |
| 集卡派 | "我想要那只 SSR" | 攒小红花召唤新只 + 图鉴陈列 |
| 召回派（忙人） | 订阅推送 → 一键回来 | 订阅消息 deeplink + 自动上行 |
| 公益本心派 | 真心想帮人 | 三 skill 主路径，Buddy 是副产物 |

---

## 2. 工程形态

**独立 skill，与 question / project / yikuaizou 并列挂载**。三个底层 SKILL.md 不动，仅在跨 skill api/call 协议表新增一行说明。

### 2.1 跨 skill 协议补丁

加在三个底层 skill 的"跨 skill api/call 协议"表中：

| 触发场景 | 上行 | 路由目标 |
|---|---|---|
| 任一成功卡返回 → 主 Agent 判 buddyMode | `getBuddySig({skillName, payloadDigest})` | **goodbuddy-skill** |
| 主业务卡 sig 区彩蛋 tap（无 Buddy 状态） | `claimBuddy({source: skillName})` | **goodbuddy-skill** |

### 2.2 主 Agent 调度逻辑（伪代码）

```
on each skill success response:
  if user.has_buddy && user.buddy_mode == ON:
    sig_payload = goodbuddy.getBuddySig(skillName, payloadDigest)
    inject sig_payload into card sig area
    LLM_bubble = sig_payload.bubble_line  # Buddy 那句话放文本气泡
  elif !user.has_buddy:
    sig_payload = {egg_hint: "你的善行可以孕育一个小伙伴 🥚 →"}
    inject into card sig area
```

---

## 3. 原子接口清单

| name | inputSchema | 返回 | 用途 |
|---|---|---|---|
| `getBuddyStatus` | （无） | `{has_buddy, current_buddy: {id, name, species, personality, rarity, ascii_frames, relation_tier}, buddies_count, mode}` | 半屏入口 / 主 Agent 路由前判定 |
| `getBuddySig` | `{skillName, payloadDigest}` | `{sig_line, sig_ascii, bubble_line, sig_arrow_target}` | 装饰底层 skill 成功卡 sig 区 + 提供文本气泡那句话 |
| `claimBuddy` | `{source?}` | `{summon_token}` → 拉起半屏 `agent_detail_pages/buddy_summon/main` | 寄生彩蛋 / 主动激活共用 |
| `nameBuddy` | `{name (≤4 字)}` | `{buddy_id}` | 命名步骤 |
| `getRecentDeeds` | `{n=5}` | `[{type, target, ts, ...}]` | Buddy 一句话生成时取材，**复用三 skill 已有数据**，不重复记账 |
| `summonNext` | （无）→ 需先校验小红花余额 | `{buddy_id}` → 拉起召唤仪式半屏 | 集卡循环 |
| `getBuddyList` | （无） | `[{id, name, species, rarity, is_current, claimed_at}]` | 图鉴 |
| `setCurrentBuddy` | `{buddy_id}` | `{ok}` | 切换当班（**5/25 不暴露 UI**，预留接口） |
| `setBuddyMode` | `{on: bool}` | `{ok}` | 开关播报 |
| `subscribeRecall` | （无）→ 拉起 `wx.requestSubscribeMessage` | `{ok}` | 订阅"明天来看 Buddy" |

---

## 4. 露出方案（路线 1：底层卡保 1:1）

**底层 skill 成功卡的卡片主体完全不动**，所有 Buddy 露出在三个位置：

### 4.1 sig 区合成（payload-side rendering）

底层 skill 成功卡的 sig 区**保留空槽**，由主 Agent 在拿到 `getBuddySig` 返回值后注入。结构：

```
sig 行 = [Buddy ASCII 小脸 4-6 字符] · [Buddy 名] · [一句话 ≤14 字] ›
```

例：`/\_/\ ( o.o ) · 白日 · 我帮你数了一下，1234 朵啦 ›`

### 4.2 文本气泡 Buddy 一句话

LLM 在 Agent bubble 里说 `getBuddySig` 返回的 `bubble_line`，**LLM 不自由发挥**——避免 5 种性格说话风格漂移。

### 4.3 半屏 Buddy 状态卡（精简版 · 见 §5）

---

## 5. 半屏信息架构（精简版，砍掉养成元素）

**砍掉的（v2 demo 半屏里有，v3 不要）**：

- ❌ 件好事 / 天陪伴 / 小红花 三大计数
- ❌ 善意值进度条
- ❌ 今日可做任务列表
- ❌ 我们一起做过的事（最近 3 条）
- ❌ 陪伴日历（本月格子图）

**v6.4 终版结构（Version A 经典竖向）**：

| # | 区块 | 内容 | 说明 |
|---|---|---|---|
| 1 | Buddy 大形象 | ASCII + 呼吸动画 + aura | tap 触发 react 表情 |
| 2 | 元信息一行 | 名字（大）+「物种 · 稀有度 · 关系称谓」（小、一行） | 关系称谓融入此行，不单独占块 |
| 3 | 任务化 quote | 5 性格各一版「再陪我做一件吧」类台词 | 始终任务驱动，不用 personality 默认招呼语 |
| 4 | **小红花进度条** | "🌸 22 / 30" + 横向进度条 + "再 8 朵可召唤下一只" + 召唤入口 | 进度 = 召唤距离，**不是今日任务完成度** |
| 5 | **任务 list（3 行）** | 简洁 3 行：✓ 捐步 / ✓ 答题 / → 捐 1 块 [去›]。已完成 line-through，未完成最后一行带操作按钮 | 无副标题、无 emoji 装饰、单行单事 |
| 6 | 底部双入口 | 图鉴 N › / 明天提醒我 ›（one-shot 订阅） | 见 §11、§8 |

**关键设计原则**：
- 一焦点：Buddy 视觉占顶部 ≈40%
- 一主行动：唯一的「去 ›」按钮在任务 list 最后一行
- 长期目标可视化：小红花进度条 → 召唤距离
- 当日具体行动：任务 list 自己显示完成度，不用额外的"2/3"进度装饰

---

## 6. 数据模型

### 6.1 Buddy 实体（user × buddy）

```
buddy {
  id: string                  # 唯一 ID
  user_id: string             # 所属用户
  species: enum               # snow_leopard / red_panda / crane / porpoise / fox
  personality: enum           # 温柔 / 可爱 / 话痨 / 神秘 / 搞笑
  rarity: enum                # N / R / SR / SSR（出生即终态）
  seed: string                # 决定 ASCII / 台词种子
  name: string                # 用户给的名字 ≤ 4 字
  claimed_at: timestamp
  interaction_count: int      # 用于关系称谓档位
  last_seen_at: timestamp
  is_current: bool            # 是否当班
}

user_meta {
  user_id: string
  has_buddy: bool
  buddy_mode: bool            # 播报开关
  current_buddy_id: string
  flower_balance: int         # 复用 project-skill 的小红花，不新增
  recall_subscribed: bool     # 是否订阅明日召回
}
```

### 6.2 行为聚合（getRecentDeeds 的数据来源）

**关键原则：GOODBUDDY 不自己重新记账，复用三 skill 已有的数据**。

- 捐款：读 project-skill 的支付订单表
- 答题：读 question-skill 的 `continuous_days` + 答题历史
- 捐步：读 yikuaizou-skill 的 fundmgr 接口

`getRecentDeeds` 在后端做一次聚合查询返回，避免数据双写不一致。

### 6.3 LLM enrich 规则（Buddy 一句话生成）

`getBuddySig` 在拼 `bubble_line` 和 `sig_line` 时：

- 30% 概率：抽 personality 默认台词库
- 30% 概率：抽 `getRecentDeeds` 近 1 件行为，LLM 改写为 Buddy 视角的回忆
- 40% 概率：拼当次行为的事实数据（比如"我帮你数过了，1234 朵啦"）

**5/25 MVP 不调外部项目接口**（如 getProjectProcess），所有 enrich 数据来自已聚合的本地数据。

---

## 7. 寄生激活路径（默认）

```
用户做好事 → 三 skill 成功卡返回
  ↓
主 Agent 调 getBuddyStatus → has_buddy = false
  ↓
sig 区注入彩蛋："你的善行可以孕育一个小伙伴 🥚 →"
  ↓
用户 tap → claimBuddy({source: 'yikuaizou'})
  ↓
拉起半屏 buddy_summon → 召唤动画（6 秒）→ 揭晓
  ↓
半屏命名步骤 → nameBuddy
  ↓
回到对话流，Agent 主动追问："要让 [白日] 帮你播报公益结果吗？"
  ↓
用户答"好" → setBuddyMode(true)
```

---

## 8. 订阅召回闭环

**这是 v3 新增的核心机制**。微信订阅消息是 **one-shot**（每授权一次只能推一次），所以 UI 入口必须是动作型而非状态切换。

**入口文案**：「明天提醒我 ›」——动作姿态，tap 后变 "✓ 明天 8 点见你"（当日不再可点）。次日若想再订阅，重新出现可点状态。


```
任一成功卡 sig 区出现「让 [白日] 明天叫你来」按钮
  ↓
用户 tap → subscribeRecall → 拉起 wx.requestSubscribeMessage
  ↓
用户授权 → 后端记录订阅
  ↓
─────── 次日早 8:00 ───────
  ↓
微信推送："白日想你了 · 今天一起做点什么？"
  ↓
用户点击通知 → deeplink 进 Agent 页面（带 scene=buddy_recall 参数）
  ↓
Agent 接收参数 → 自动模拟用户上行预设消息："我来看看 buddy"
  ↓
LLM 路由到 goodbuddy::getBuddyStatus → 返回 1:1 Buddy 召回卡
  ↓
卡内：Buddy ASCII + 当下一句话 + 3 件可做的好事入口（路由到三 skill）
```

### 8.1 工程依赖待确认（袁康）

- 微信 Agent 是否支持 deeplink 参数透传？
- Agent 自动上行消息能力是否就绪？
- 订阅消息模板：复用 `STEP_REMINDER_TPL_ID` 改文案，或单独申请 `BUDDY_RECALL_TPL_ID`

**降级方案**：deeplink / 自动上行不支持时，订阅 push 只跳小程序主页，Buddy 主动 say hi。体验打折但不致命。

---

## 9. 关系演进（关系派增益）

### 9.1 关系称谓 4 档查表

| `interaction_count` | Buddy 称呼 | 语气基调 |
|---|---|---|
| 1-2 | "你" / "您" | 礼貌、初识 |
| 3-5 | "嗨" / 打招呼 | 熟络 |
| 6-10 | "老朋友" | 自然 |
| 10+ | 用户的微信昵称 | 亲昵 |

每档对应的台词库由文案侧维护，GOODBUDDY skill 内置。

### 9.2 Buddy 偶尔提及近期行为

详见 §6.3 LLM enrich 规则。**5/25 MVP 只做"提及近 1 件本地行为"**，不调外部项目接口。

---

## 10. 召唤新 Buddy（集卡派）

- **门槛**：30 朵小红花（后端 config 化，起步参数）
- **流程**：用户在半屏 tap"召唤下一只" → 校验小红花 → 拉起召唤仪式（复用 demo 现有半屏 `hs-rite` 视图）→ 揭晓 → 进入 Buddy 列表
- **稀有度概率**：N 60% / R 25% / SR 12% / SSR 3%
- **5/25 不暴露**：切换当班 UI（接口预留）、多 Buddy 共存场景

---

## 11. 多 Buddy 数据位 + 图鉴

- **后端**：从一开始就是 `buddies[]` 列表结构 + `current_buddy_id` 指针
- **5/25 前端只露**：当班 Buddy（播报、半屏顶部大形象）+ 一个"我的图鉴"半屏 tab
- **图鉴**：所有召唤过的 Buddy 以缩略 ASCII 卡形态陈列，tap 可看完整形象 + 当时台词 + 稀有度。**未当班 Buddy 暂不可切换**（接口预留，UI 二期）
- **未拥有 Buddy**：图鉴里灰色占位，写"再做 N 件好事可召唤"——双向承接"已拥有/未拥有"视图，强化集卡心理

---

## 12. 5/25 MVP 范围 vs 二期清单

### 5/25 必做

- [x] 寄生激活彩蛋（三 skill 成功卡 sig 区注入）
- [x] 主动激活路径（用户主动说"想要 buddy" → Agent 引导做一件好事 → 进入认领）
- [x] 即时召唤 + 揭晓动画 + 稀有度
- [x] 5 动物 × 5 性格 × 4 稀有度 ASCII + 完整台词库（出生即全开放）
- [x] 命名步骤
- [x] sig 区合成 + 文本气泡 Buddy 一句话
- [x] 半屏精简版（§5）
- [x] **今日陪 Buddy 做 · 任务制喂养（v6.1 加回）**——3 件任务对应底层三 skill，完成 → Buddy 吃花 + 一句反应。**不引入新货币、不引入养成深度，是日常入口聚合 + 仪式感包装**
- [x] 关系称谓 4 档查表
- [x] Buddy 偶尔提及近 1 件本地行为
- [x] 召唤新 Buddy（小红花门槛 30 朵起步）
- [x] 多 Buddy 数据位 + 图鉴（v6.1 缩小为单行小入口）
- [x] 订阅召回（degraded 版本可接受）
- [x] 分享卡（demo Scene 3 的 5 张设计，揭晓后 prompt 保存）

### 二期（6 月后）

- [ ] 调外部 project 接口查最新进展，Buddy 提及"项目又有动静"
- [ ] 切换当班 UI
- [ ] 多 Buddy 共存场景（一个卡里两只）
- [ ] 台词扩展 / 主题广度解锁
- [ ] 小成就标记（曾陪伴 N 件好事的小星星）
- [ ] 朋友圈分享时挑选哪只 Buddy
- [ ] Buddy 在微信状态里的呈现（戴 Buddy 头像而非小红花）

---

## 13. 工程依赖待确认

| # | 依赖 | 确认对象 | 阻塞性 |
|---|---|---|---|
| 1 | sig 区可由其他 skill 注入 payload（payload-side rendering） | 袁康 | **5/25 阻塞** |
| 2 | deeplink 参数 + 自动上行消息能力 | 袁康 | 中（有降级方案） |
| 3 | 订阅消息模板 ID（复用 vs 新申请） | 方老师 | 低 |
| 4 | 小红花余额接口（GOODBUDDY 要读，project-skill 现有？） | 袁康 | **5/25 阻塞** |
| 5 | 三 skill 的行为聚合接口（getRecentDeeds 的后端实现） | 袁康 | 中 |

---

## 14. 跟 v2 核心宪法的差异（明确记录）

v2 31 号文件 → v3 47 号文件，主要覆盖以下决策：

- **v2: 10 天孵化** → v3: 即时出生（孵化作为机制下放到二期"召唤"，但 v3 暂不引入）
- **v2: 善意值经济系统** → v3: 砍掉，统一小红花
- **v2: 蚂蚁森林化养成（喂养 → 长大）** → v3: 赠送品 + 关系演进
- **v2: Buddy 是核心串联（隐含养成深度）** → v3: Buddy 是主链路（强调第一印象 + 分享性 + 订阅召回）
- **v2: 5/25 MVP = 认领 + 孵化 + 出生 + 1 个喂养行为** → v3: 5/25 MVP = 认领 + 即时出生 + 关系演进 + 订阅召回 + 集卡

**保留不变**：真功德原则、接地气文案基调、不引设计师、护城河逻辑（项目网络 + 微信运动数据 + 支付小红花 + IP + 串联）

---

## 15. 待办清单（你与方老师 / 袁康对齐时用）

**产品侧（与方老师）**：
1. v2 → v3 哲学转向是否买账（赠送品 vs 养成品）
2. 关系称谓 4 档的具体文案库谁来产出
3. 5 性格 × 5 物种的台词库扩展到多少条够 MVP（建议每只每场景 5-8 条）
4. 召唤门槛 30 朵小红花是否合理（参考公益捐步 DAU 177 万的常见用户小红花获取节奏）

**工程侧（与袁康）**：
1. §13 表格中阻塞性项目逐一确认
2. sig 区 payload-side rendering 技术方案
3. 跨 skill api/call `getBuddySig` 路由路径
4. Buddy 实体 + 行为聚合的后端表设计
5. 5/25 之前的接口联调时间表

**视觉侧（无设计师，手搓）**：
1. 5 动物 ASCII 已有（demo 现状）
2. 命名步骤 / 揭晓页 / 半屏精简版 / 图鉴 视觉一致性
3. 分享卡（Scene 3）质感是否再迭代一版（这是核心传播器）

---

*本文档作为 5/25 GOODBUDDY skill 的工程对齐基线，后续修改请通过新版本号（v3.1 / v4）追加，不直接覆盖。*
