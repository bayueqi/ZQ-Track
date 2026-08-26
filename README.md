# ZQ-Track订阅管理与提醒系统

基于 **Cloudflare Workers + KV** 的轻量级订阅到期提醒。单文件部署，无需服务器，在网页里管理订阅，到点通过 Telegram / Bark / 企业微信 / ntfy / 邮件 / Webhook 等 **10 种渠道** 推送，并自带发送与调度日志方便排查。

**适合**：个人自托管、域名/会员/账单到期提醒、云端零成本运行。
**不适合**：多用户协作、复杂企业审批流。

---

## ✨ 特性

- 🚀 **零服务器**：完全运行在 Cloudflare Workers 边缘网络
- � **单文件部署**：只需一个 `worker.js`，无需复杂构建
- 🔔 **10+ 通知渠道**：Telegram、Bark、企业微信、ntfy、Server酱、PushPlus、Gotify、NotifyX、邮件(Resend)、Webhook
- 🌙 **暗黑模式**：自动跟随系统/手动切换
- 🧧 **农历支持**：支持农历周期的订阅（如按月/按年续费的农历服务）
- 💰 **多币种**：自动汇率转换与支出统计
- 📊 **仪表盘**：可视化订阅支出与到期分布
- 📜 **完整日志**：通知历史、调度记录、失败原因一目了然
- 🔄 **自动续订**：支持循环订阅和到期重置两种模式
- 💾 **备份导入导出**：JSON 格式方便迁移

---

## 📦 快速部署（单文件版）

### 准备工作

1. 一个 Cloudflare 账号（免费版即可）
2. 本仓库提供的 `worker.js` 文件

### 方式一：Cloudflare Dashboard 可视化部署（最简单，推荐新手）

**步骤 1：创建 KV 命名空间**
1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 左侧菜单 → **Workers & Pages** → **KV**
3. 点击 **Create a namespace**
4. 名称填写 `Track`（或任意你喜欢的名字），点击 **Add**

**步骤 2：创建 Worker**
1. 左侧菜单 → **Workers & Pages** → **Create application** → **Create Worker**
2. 给你的 Worker 起个名字（如 `subs-tracker`），点击 **Deploy**
3. 部署完成后，点击 **Edit code**（快速编辑）
4. 删除编辑器里的所有默认代码，将 `worker.js` 文件的**全部内容**粘贴进去
5. 点击右上角 **Deploy** 保存

**步骤 3：绑定 KV**
1. 回到 Worker 页面，点击 **Settings** 标签
2. 找到 **Variables** 区域
3. 在 **KV Namespace Bindings** 下点击 **Add binding**
   - Variable name：`Track`（必须完全一致）
   - KV namespace：选择你刚才创建的 KV 命名空间
4. 点击 **Save** 保存

**步骤 4：配置定时触发器（重要！用于自动检查）**
1. 还是在 Settings 页面，找到 **Triggers** 标签
2. 找到 **Cron Triggers** 区域，点击 **Add Cron Trigger**
3. 在 Cron expression 填写：`0 * * * *`（每小时整点执行一次，免费版足够用）
4. 点击 **Add Trigger**

**🎉 完成！** 访问你的 Worker 地址（类似 `https://subs-tracker.你的用户名.workers.dev`）即可开始使用。

---

### 方式二：使用 Wrangler CLI 部署

如果你熟悉命令行，可以用 Wrangler 一键部署：

1. 安装 Wrangler：
```bash
npm install -g wrangler
```

2. 登录 Cloudflare：
```bash
wrangler login
```

3. 在 `worker.js` 同目录下创建 `wrangler.toml`：
```toml
name = "subs-tracker"
main = "worker.js"
compatibility_date = "2024-09-23"
compatibility_flags = ["nodejs_compat"]

# KV 命名空间（第一次部署请先运行下面的创建命令）
[[kv_namespaces]]
binding = "Track"
id = "运行创建命令后这里会自动填充"

# 每小时定时检查
[triggers]
crons = ["0 * * * *"]
```

4. 创建 KV 命名空间：
```bash
wrangler kv:namespace create Track
```
将输出的 `id` 填入 `wrangler.toml` 对应位置。

5. 部署：
```bash
wrangler deploy
```

---

## 🔑 默认登录

| 项 | 值 |
|----|-----|
| 用户名 | `admin` |
| 密码 | `password` |

⚠️ **登录后请立刻在「系统配置」中修改密码！** 公网暴露默认密码非常危险。

### 忘记密码怎么办

Cloudflare Dashboard → **Workers & Pages** → **KV** → 打开你绑定的 KV → 找到 key 为 `config` 的条目，编辑 JSON，修改 `ADMIN_PASSWORD` 字段后保存即可。

---

## ⚙️ 第一次必做配置

部署完成登录后，打开 **系统配置** 页面，按顺序完成以下设置：

### 1. 修改管理员密码
第一件事！把默认的 `admin/password` 改成你自己的。

### 2. 设置时区
- **中国大陆用户选 `Asia/Shanghai`（北京时间）**
- 所有到期时间、提醒时间都以此时区为准

### 3. 配置允许发送通知的小时
| 配置 | 含义 |
|------|------|
| `08` | 只在北京时间早上 8 点左右发通知 |
| `08, 20` | 早 8 点、晚 8 点各一次 |
| 留空或 `*` | 每小时都可以发（不推荐，可能太频繁）|

💡 提示：定时任务每小时整点运行，但只有在你配置的小时内才会实际发送通知，避免半夜打扰。

### 4. 配置至少一种通知渠道
勾选你想用的渠道，填写 Token/密钥，点击「测试通知」按钮，确认能收到消息后保存。

**常用渠道配置指引：**
| 渠道 | 需要准备 |
|------|----------|
| Telegram | 找 @BotFather 创建机器人拿到 Bot Token，然后给机器人发一条消息，再获取你的 Chat ID |
| Bark | iOS 安装 Bark App 拿到设备 Key，自建服务可填服务器地址 |
| 企业微信 | 群聊里添加群机器人，复制 Webhook 地址 |
| ntfy | 手机安装 ntfy App，订阅一个 Topic（默认用 ntfy.sh 公共服务器）|
| 邮件 | 去 resend.com 注册拿 API Key（免费额度足够个人用）|
| Webhook | 任意可以接收 POST 请求的 HTTP 地址 |
| Server酱/PushPlus/NotifyX/Gotify | 去对应平台注册拿到密钥即可 |

### 5. 添加一条测试订阅
- 到期日设为今天或明天
- 提醒规则选「应用预设 7/3/1/当天」
- 或者临时把「允许发送的小时」设为当前小时，测试通知是否正常发送

---

## � 使用指南

### 订阅管理
- **添加/编辑**：名称、金额、分类、周期、到期日、备注
- **克隆**：快速复制类似订阅（适合多张保号卡这类）
- **续订**：手动延长到期日并记录支付
- **停用/启用**：暂时不用的订阅可以停用，不会再发提醒
- **测试通知**：立即给该订阅发一条测试消息
- **支付历史**：查看每次续费记录
- **筛选**：按关键词、状态、分类筛选订阅

### 提醒规则说明
默认预设：**到期前 7 天、3 天、1 天 + 到期当天** 发送提醒。

⚠️ **重要：**「到期前 N 天」是指 **剩余天数正好等于 N 的那一天只发一次**，不是从第 N 天开始每天发。
如果需要 7、6、5...天都提醒，需要手动添加多条规则。

规则类型：
- **到期前 N 天**：精确匹配天数
- **到期当天**：到期日当天发
- **到期后每 N 小时**：过期后持续提醒直到你续费

### 订阅模式
| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **循环订阅** | 续费时从**当前到期日**往后顺延一个周期 | 视频会员这类，提前续费也不会亏时间 |
| **到期重置** | 续费时从**支付当天**重新计算周期 | 手机保号卡这类，充值后从充值日重新算有效期 |

### 备份与迁移
在系统配置页面最下方：
- **导出备份**：下载所有数据为 JSON 文件
- **导入备份**：
  - 合并模式：按 ID 覆盖已有订阅，保留其他数据
  - 覆盖模式：清空所有数据再导入（⚠️ 危险，先导出当前备份）

---

## ❓ 常见问题

### 为什么没收到通知？
按顺序排查：
1. 通知渠道是否启用？测试通知能不能收到？
2. 「允许发送的小时」是否包含当前小时？
3. 时区是否设置正确（国内用户务必是 `Asia/Shanghai`）？
4. 订阅是否启用？提醒规则是否启用？
5. 今天是否正好命中提醒天数？（注意是「正好 N 天」不是「N 天以内」）
6. 去「通知历史」页面看是跳过了还是发送失败了，失败会有具体错误原因。

### 通知历史显示「不在允许发送的小时」？
这是正常的！说明定时任务跑了，但你配置了只在特定小时发，所以跳过了。等你设置的小时到了自然会发。

### 支持绑定自定义域名吗？
支持！在 Cloudflare Worker 的 Settings → Triggers → Custom Domains 里添加你的域名即可。

### 免费版 Cloudflare 够用吗？
完全够用。免费版 Workers 每天 10 万次请求，KV 每天 10 万次读取，对于个人订阅管理来说绰绰有余。

### 数据存在哪里？安全吗？
所有数据都存在你自己的 Cloudflare KV 存储中，不会经过第三方服务器。记得保管好你的 Cloudflare 账号。

### 怎么升级到新版本？
直接下载新的 `worker.js`，在 Cloudflare Dashboard 的 Worker 编辑器里全选替换内容，点 Deploy 即可。数据在 KV 里不会丢失，大版本升级建议先导出备份。

---

## 🛠 本地开发（开发者）

如果你想修改代码：

```bash
# 安装依赖
npm install

# 本地开发
npx wrangler dev --config wrangler.dev.toml --local

# 运行测试
npm test

# 重新打包为单文件
node build.cjs
```

项目结构：
```
src/
├── index.js           # Worker 入口（fetch + scheduled）
├── app.js             # Hono 路由
├── core/              # 时间计算、农历、货币、JWT
├── data/              # KV 操作和数据迁移
├── services/          # 调度器和各个通知渠道实现
├── api/               # API 路由处理器
├── views/             # 前端 HTML 页面
└── static.js          # 内联静态资源
```

---

## 🔐 安全建议

1. **部署后立刻修改默认密码**
2. 不要把 Cloudflare API Token、各种通知渠道的 Token 提交到公开仓库
3. 备份的 JSON 文件如果包含敏感配置，请妥善保管
4. 如果怀疑 Token 泄露，及时去对应平台轮换密钥

---

## 📄 开源协议

MIT License - 详见 LICENSE 文件。

---

## 致谢

- 基于 [Hono](https://hono.dev/) 超快的 Web 框架
- 农历算法参考了开源实现
- 感谢 Cloudflare 提供优秀的边缘计算平台
