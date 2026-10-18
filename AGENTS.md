# OrderFile · 客户订单信息流系统

> **一个 projNo，一条谁都不会覆盖的时间轴。**

## 🎯 项目定位

**OrderFile** 不是 CRM，不是客户管理。是一个**以工程单号 `projNo` 为主键的信息流水账**。

**唯一的立项原因**（2026-10-14 Sami 原话）：

> 业务员接了单，后勤人员对客户不了解。为了解决沟通成本、避免出错，才需要这个系统。

角色链路：**业务员 → 设计师 → 下单员 → 安装员**。
四个角色往**同一张单**上追加记录，谁都不覆盖谁。安装师傅打开这一单，
立刻知道这个客户是谁、什么脾气、之前说过什么、有什么禁忌。

**核心命题：客户三个月前说的话，还在。**

---

## 🧭 与现有系统的边界（重要）

OrderFile **独立运行，不修改任何现有系统**。它**只读** Squirrel Designer 的报价数据。

| 系统 | 关系 | 说明 |
|---|---|---|
| Squirrel Designer (29) | **只读** | 提供 projNo / 客户名 / 电话 / 地址 / 金额 / 交期 |
| backadmin (30) | 无关 | 不交互 |
| SquirrelInstall (31) | **平行** | 柜子物流视角（装柜/抵达）。OrderFile 是客户视角。**两者不合并** |
| SquirrelFinance (32) | 无关 | 不交互 |
| Designerproducts (27) | 无关 | 产品库 |

### 为什么不合并 SquirrelInstall（2026-10-14 决定）

1. **视角不同** — SquirrelInstall 记录「柜子走到哪了」，OrderFile 记录「客户说过什么」。
   塞一个页面里两边都别扭。
2. **会推翻 v3 数据模型** — SquirrelInstall 的 `stage` + `stages.rows[]` 是覆盖写模型，
   而 OrderFile 的核心是 append-only 时间轴。强行合并要重写它的数据模型和迁移逻辑。
3. **OrderFile 不依赖 SquirrelInstall 存在** — 哪天柜子追踪停了，订单档案还得继续用。

---

## 📦 数据存储

```
Google Drive:
  OrderFile/
    config.json                    ← 人员 + 角色配置（admin 可在界面改）
    archives/
      PSD01-2401.json              ← 一个订单 = 一个档案文件
      PSD01-2402.json
    attachments/
      PSD01-2401/
        20261018_PEGGY_客户签字确认单_V3.jpg
        20261012_PEGGY_现场.jpg

Squirrel Designer (只读):
  Squirrel Designer/{username}/{projNo}.json
```

### 为什么不把客户资料抄进来

报价会改、金额会变、电话会换、客户会改名。**一旦复制就开始对不上，半年后没人知道信谁。**

OrderFile **只存它没有的东西**：画像补充、硬性要求、设计版本、问题、变更、验收、时间轴。
客户名/电话/地址/金额/交期全部实时读 Squirrel Designer。

唯一例外：`customerSnapshot` 在**确认签字那一刻**冻结一份（见下）。

---

## 🧬 数据模型

```js
// archives/{projNo}.json
{
  projNo: "PSD01-2401",
  createdAt, updatedAt,
  currentOwner: "sales",          // sales|design|order|install|done —— 算出来的，不是拖出来的
  ownerSince: "2026-10-13T10:02", // 用来算逾期天数

  // 业务员填的画像补充（chips，不覆盖报价系统）
  profile: {
    chips: ["洁癖","不能抽烟","有小孩"],   // 自由 chips + 预设清单
    contactPref: "微信",                    // 沟通偏好
    decisionMaker: "",                     // 决策人
    taboo: "",                             // 禁忌（自由文字）
  },

  // 硬性要求：红色 = 必须做
  requirements: [
    { id, text, priority: "必须做"|"尽量做"|"已拒绝", createdBy, createdAt }
  ],

  // 确认件（唯一的法律证据）
  confirmations: [
    {
      version: "V3",
      method: "signed"|"written",     // signed=有签字件 / written=只有截图
      amount: 38600,                  // 生成确认单那一刻的金额，冻结
      deliveryDate: "2026-11-15",
      confirmedAt, confirmedBy,
      evidence: [ { name, url, size } ],   // 签字件照片 / WhatsApp 截图
      locked: true,
      quoteHash: "…"                   // 报价快照哈希，金额被改时用来检测失效
    }
  ],

  // 未读追踪
  lastRead: { "PSD": "2026-10-12T09:00", "VSD": "…" },

  // 时间轴：只追加，永不覆盖
  events: [
    {
      id, at,                        // at 用服务端时间，不信客户端
      author,                        // username
      role,                          // sales|design|order|install
      type,                          // 要求|问题|注意事项|沟通|方案|确认|变更|现场问题|排期|退回
      priority,                      // 必须做|尽量做|已拒绝|""
      content,
      photos: [ { name, url, size } ],
      docs:   [ { name, url, size } ],
      assignTo: "STELLA"|null,
      handoff: { to, at }|null
    }
  ]
}
```

### 三个铁律

1. **events 只追加，永不 update。** 这是整套系统的地基 —— 现有 6 个系统全是覆盖写，所以「谁改了什么」永远查不到。
2. **确认件锁定不可删、不可替换。** 要改只能走「重新确认 V4」，旧件留在时间轴上。
3. **currentOwner 由事件推导，不由人拖拽。** 手拖的 stage 一定会跟现实脱节。

---

## 🔄 流转规则

```
业务员 ──画像+报价──► 设计师 ──方案 Vn──► 业务员 ──客户签字──► 下单员
                                                              │
                                          ┌───────────────────┴──────────────┐
                                          ▼（走柜）                            ▼（不走柜）
                                        物流 ──装柜/抵达──► 安装员 ──验收──► 完结
                                                                          ▲
                                          物流 ───────────────► 安装员 ─────┘
```

| 谁提交 | 提交什么 | 交接给 |
|---|---|---|
| 业务员 | 客户画像 + 报价链接 + 现场照片 | 设计师 |
| 设计师 | 量尺 + 方案 Vn + **待确认稿（必填）** | 业务员（去约客户签字） |
| 业务员 | **「客户已签字 Vn」+ 签字件照片（必传，按钮点不动）** | 下单员 |
| 下单员 | 交期 + 包装要求 + 给工厂的特殊工艺说明 | **物流（走柜）** 或 安装（不走柜） |
| 物流 | 发货 / 装柜 / 抵达 / 物流异常 | 安装员 |
| 安装员 | 现场照片 + 安装问题 + 验收 | 完结 |
| 任何人 | 「退回上一环」，退回本身也在时间轴留痕 | 上一环 |

### 下单员可以跳过物流

不是所有单都走柜（本地送货）。所以 `order` 有两个交接目标：
`logistics`（走柜）或 `install`（直接装）。**其他环节不可跳** —— 这是链路上唯一的分叉点。

### 「必须传签字件」是唯一强制卡点

理由：其他环节漏填，后面的人问一句就补上了。
但签字件没传，**下单员就没有合法依据去下单**；客户翻脸说「我签的不是这一版」，没人拿得出版本证据。

**没上传附件，「提交并交接给下单员」这个按钮就是灰的，点不动。**

### 客户不签字的出口

不设死路，但**证据等级不同**：

| 出口 | 必传 | 订单标记 |
|---|---|---|
| 客户已签字（优先） | 签字件照片 | `已确认 V3` + 锁定 |
| 客户已书面确认（备用） | WhatsApp 截图 / 邮件 | `未经签字确认`（全员可见） |

---

## 👥 人员名册：不在本系统维护

**真相源 = `Squirrel Designer/users.json` 的 `departments` 字段**（主系统）。
本系统只读，不写、不复制。

```
Squirrel Designer/users.json          ← 唯一真相源
        │ departments
        ▼
DEPT_ROLE_MAP 映射
        │ + OrderFile/config.json 的 roleOverrides（只存偏离项）
        ▼
登录页名单 + 顶栏角色下拉
```

### 部门名映射（⚠️ 主系统的叫法，不是猜的）

| 主系统 `departments` | 本系统 role | 中文 |
|---|---|---|
| `admin` | `admin` | 管理员 |
| `sales` | `sales` | 业务员 |
| `designer` | `design` | 设计师 |
| `orderClerk` | `order` | 下单员 |
| `install` | `install` | 安装员 |
| `logistics` | `logistics` | 物流 |

> 坑：主系统叫 `designer` 和 `orderClerk`，不是 `design` / `order`。
> 映射表已处理，但**新增部门时别忘了同步这张表**。

### roleOverrides 只存「偏离项」

`saveConfig` 会把提交的角色跟主系统基线**逐个比对**，只有真的不一样的才写进 `roleOverrides`。
如果把整份名册都存进去，等于把名册冻住 —— 以后主系统加/减人，OrderFile 就不跟着变了。

一人可以有多个角色（如 VSD JOVEN = designer + sales）。
登录后自己在顶栏下拉切换当前角色。

**改角色的推荐做法**：在主系统改 `departments`（backadmin → 用户管理），
OrderFile 自动同步。只有主系统表达不了的才用本系统的「角色设置」页覆盖。

---

## 📄 确认单（签字件）

**系统生成 A4 PDF，业务员打印带去，客户签字，拍照回传。** 不手打。

必含字段：projNo / 客户姓名 / 安装地址 / **确认版本** / 确认日期 /
**合同金额** / **交期** / 客户签名 / 业务员签名。

### 金额自动带出并冻结 —— 最关键的一行

金额、交期、客户名、地址全部从 Squirrel Designer 实时读，生成确认单时**冻结进 `confirmations[].amount`**。

理由：「客户签的是哪个价」是纠纷最高频的起因。

### 金额变更 → 旧签字件自动作废

系统比对 `confirmations[].amount` 与当前报价金额，不一致就在订单上打红标：

> **金额已变更，需重新签字**

逼着人走流程，而不是让问题烂在微信里。同一逻辑适用交期、方案版本。

### 防倒签

`confirmedAt` 用**服务端时间**，绝不信客户端手机时间（手机时间可改）。

---

## 🔔 站内提醒（三处同时出现）

提醒最怕的不是没做，是**「提醒了但没人点」**。所以必须同时出现在三处：

1. **首页顶部** —「待我处理 N」红色数字
2. **订单列表旁** — 红点 +「N 条新」
3. **订单详情里** — 时间轴**未读分隔线**「你上次看到这里」

### 双向提醒

| 方向 | 他想知道什么 | 显示在哪 |
|---|---|---|
| 别人 → 我 | 有没有新的事要我处理 | 首页待我处理 + 列表红点 |
| **我 → 我** | **我提交的东西有回应了没** | 「我的提交」列表 + 状态变化 |

第二个方向最容易漏，但恰好就是「沟通成本」。设计师提交方案后最想知道「客户确认了没」。

---

## 🛡 五道防线（避免出错）

| | 防线 | 防什么 |
|---|---|---|
| ① | 硬性要求红色置顶 | 漏做 |
| ② | 未读分隔线 | 漏看 |
| ③ | 确认必须是点出来的按钮 | 抢跑 |
| ④ | 签字件照片必传 | 无依据下单 + 金额争议无证据 |
| ⑤ | 能退回上一环 + 留痕 | 错了拉不回 |

---

## 📱 手机端

**PWA，不做原生 App。** 装到主屏幕后有图标、全屏打开。

### 安装师傅 3 步

**点桌面图标 → 按绿色大按钮（拍照）→ 点完成**

原则：**不让他打字。**

- 现场问题用 chips 点选（墙面不平 / 缺件 / 磕碰 / 门缝不对 / 尺寸有误 / 客户改口），覆盖 90% 场景
- 剩下 10% 才给输入框，**而且可以留空**
- 装完 3 张照片比写一段话有价值得多 —— **沟通成本本来就该由系统吃掉**

### 业务员（站立式表单）

顺序：projNo → chips → 内容输入框 → 拍照 → 指派。
拍照按钮一个就够 —— 签字单和现场照片共用同一个。

### 技术细节（第一版就要做对）

| | 规则 | 原因 |
|---|---|---|
| 压缩 | 前端压到长边 1600px / JPEG 0.75 | 原图 3-5MB 直传要 30 秒+，传一半人走了 |
| 命名 | `{projNo}/{日期}_{角色}_{简述}.jpg`，**用户不能改** | 否则三个月后全是 IMG_4821.jpg |
| 分类 | 按登录身份**自动定**，不让用户选 | 让人选会导致业务员选「设计」、设计师选「下单」，时间轴串位 |
| 指派 | 用户可选「指派给谁跟进」 | 这是任务不是分类 |

---

## 🔌 GAS Action 清单

| Action | 用途 |
|---|---|
| `health` | 健康检查，返回 version |
| `getConfig` | 读人员 + 角色配置 |
| `saveConfig` | admin 保存角色配置 |
| `getQuoteInfo` | **只读** Squirrel Designer，返客户资料 + 金额 |
| `listOrders` | 订单摘要列表（currentOwner + 未读数 + 逾期天数） |
| `searchOrders` | projNo / 客户名 / 电话 / 关键词 |
| `getOrder` | 单个订单完整档案 |
| `createOrder` | 按 projNo 建档案（先校验 quote 存在） |
| `appendEvent` | **追加一条记录**（核心）+ 自动 handoff + 写时间戳 |
| `addRequirement` | 加硬性要求 |
| `submitConfirmation` | **提交确认件**（校验签字件存在 + 版本号 + 冻结金额） |
| `invalidateConfirmations` | 金额/交期变了 → 标记旧确认件失效 |
| `markRead` | 更新 lastRead |
| `getMyTasks` | 待我处理 + 我的提交（双向提醒） |
| `uploadFile` | base64 → Drive 附件 |
| `deleteOrder` | 移到回收站（不物理删） |

### 传输约定

- **GET** 用于读，`?action=xxx&k=v`
- **POST** 用 `application/x-www-form-urlencoded`（GAS 只读 `e.parameter`）
  —— **不要用 `application/json`**，会触发 CORS 预检导致浏览器端静默失败
- `at` / `confirmedAt` 一律服务端 `new Date().toISOString()`

---

## 📁 文件结构

```
projects/38/orderfile/
├── AGENTS.md              ← 本文件
├── README.md              ← 使用说明（给一线看的）
├── docs/DEPLOY.md         ← 部署指南
├── gas/Code.gs            ← GAS 后端源码
└── dist/
    ├── index.html         ← 前端单文件 SPA
    ├── manifest.webmanifest
    ├── sw.js
    └── icon-192.png / icon-512.png
```

---

## 🚀 部署流程

见 `docs/DEPLOY.md`。要点：

1. 用户在 https://script.google.com/home 新建 GAS 项目 `OrderFile`
2. 粘贴 `gas/Code.gs` → 保存
3. 部署 → **新部署** → Web 应用 → 执行身份「我」→ 访问「任何人」→ 复制 URL
4. 把 URL 填到 `dist/index.html` 的 `GAS_URL` 常量
5. 用户建 GitHub 仓库 `orderfile`（Public），agent push
6. GitHub Pages：Source = main / (root)

### appsscript.json

```json
{
  "oauthScopes": [
    "https://www.googleapis.com/auth/drive",           // 不能用 drive.file
    "https://www.googleapis.com/auth/script.external_request"
  ]
}
```

### Squirrel Designer GAS URL（只读源）

`https://script.google.com/macros/s/AKfycbyOEmxMojsICWRgpLgII-fB1jniWTCMLBMSvwFUxAz6IhpZsdRnMRfENV2p88LOQ7cm/exec`

> ⚠️ 改前端或换部署后**从前端 HTML 里重新提取**这串 URL，不要凭记忆写。
> 用 `[regex]::Matches($c,'https://script\.google\.com/macros/s/[A-Za-z0-9_\-]+/exec')`

---

## 🔮 v2 待办（第一版明确不做）

- iPad 手写签名层（离线打开方案 → 手写 → 同步）
- 企业微信 / 钉钉 / 短信外部通知（站内标识已够用）
- 权限脱敏（内网先用着）
- 报表 / 统计
- 酷家乐 / 拆单软件 / ERP 对接
- 原生 App
