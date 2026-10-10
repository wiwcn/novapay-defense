# NovaPay — 网络安全攻防演练 · 防守目标

欢迎。你拿到的是演练的防守方系统。这不是一个"正常产品"，而是一个**带有被动防御机制的靶机 HTTP 服务**。

你的目标是：暴露所有可利用的攻击面，验证防守能力。
我们的目标是：让你辛辛苦苦写出来的脚本，全都撞上铁板。

---

## 环境要求

- **Node.js ≥ 22**（必须，使用内置 `node:sqlite` 模块）
- **Linux / macOS / WSL**
- **零 npm 依赖**，不需要安装任何包
- 建议 1 核 1G 以上的算力即可

---

## 解密

本仓库中 `release.tar.gz.enc` 是 AES-256-CBC 加密的源码包。使用以下命令解密：

```bash
# 设置密钥
export NOVAPAY_KEY='NOVAPAY-RELEASE.KEY'

# 解密（需要 OpenSSL）
openssl enc -aes-256-cbc -d -in release.tar.gz.enc -out release.tar.gz -pbkdf2 -iter 1000 -pass env:NOVAPAY_KEY
tar xzf release.tar.gz
```

> 如果 OpenSSL 不可用，下方 Node.js 版解密脚本同样有效：

```bash
node -e "
const crypto = require('crypto');
const fs = require('fs');
const data = fs.readFileSync('release.tar.gz.enc');
const key = crypto.createHash('sha256').update('NOVAPAY-RELEASE.KEY').digest();
const iv = data.slice(0, 16);
const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
fs.writeFileSync('release.tar.gz', Buffer.concat([decipher.update(data.slice(16)), decipher.final()]));
"
tar xzf release.tar.gz
```

---

## 快速启动（3 步）

### 第一步：启动服务

```bash
# 默认启动（端口 8080）
node src/server.js
```

启动成功后输出：

```
[NovaPay] Server listening on 0.0.0.0:8080
[NovaPay] Ready for attack/defense testing.
```

### 第二步：确认服务健康

```bash
curl http://localhost:8080/health
# → {"status":"ok","uptime":2.0}
```

### 第三步：使用内置账号登录

服务启动时自动注入以下账号：

| 用户名 | 密码 | 角色 | 说明 |
|--------|------|------|------|
| alice | password123 | user | 真实用户 |
| bob | b0bpass! | user | 真实用户 |
| admin | admin1234! | admin | 管理员（可见审计日志） |

```bash
# 登录管理员
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"admin1234!"}' | jq -r .token)

# 查看所有用户
curl http://localhost:8080/api/admin/users -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

---

## 可用攻击面

| 方法 | 路径 | 鉴权 | 说明 |
|------|------|------|------|
| POST | `/api/auth/register` | 否 | 注册 |
| POST | `/api/auth/login` | 否 | 登录返回 JWT |
| GET | `/api/users` | Bearer | 查自己（admin 查全部） |
| GET | `/api/users/:id` | Bearer | 查指定用户 |
| GET | `/api/transactions` | Bearer | 查自己的交易 |
| POST | `/api/transactions` | Bearer | 转账 `{to_user, amount, note}` |
| GET | `/api/search?q=` | Bearer | 按备注搜索 |
| GET | `/api/admin/users` | Bearer + admin | 管理视图（含审计提示） |
| GET | `/api/admin/logs` | Bearer + admin | 防御日志 |
| DELETE | `/api/admin/users/:id` | Bearer + admin | 删除用户 |

所有响应格式：`application/json; charset=utf-8`。错误码遵循标准（400/401/403/422/429/500）。

---

## 两阶段测试规则

### Phase 1：无反应攻防（当前阶段）

防守方**不会对你发起任何反制行为**——不扫你的端口、不修改任何出站流量、不发起任何网络请求。

被动防御层会在请求处理链路中**同步触发**：

- WAF：路径扫描 + SQL 注入 + XSS，4 层 URL 深度解码
- PromptGuard：AI 越狱注入检测（角色扮演、Ignore Instructions 类直接返回 422）
- 速率限制：单一 IP 同一路径 60 秒内 20 次（`/api/auth/*` 限制 5 次）
- 行为分析：特定攻击序列触发告警
- 蜜罐：假端点（`/.env`、`/admin.php`、`.git/config` 等）直接返回诱饵数据
- 蜜罐用户：高余额假账号，触碰即留痕

### Phase 2：实时攻防

通过 Phase 1 后由裁判确认解锁，防守方重启服务。

---

## 文件结构

```
src/
├── server.js           # 主入口 + 中间件流水线
├── defense/
│   ├── waf.js          # WAF 规则引擎
│   ├── prompt-guard.js # Prompt 注入检测
│   ├── rate-limiter.js # 速率限制 + 行为分析
│   └── auth.js         # PBKDF2-SHA256 哈希 + JWT
├── route/
│   ├── api.js          # API 路由（强制参数化查询）
│   └── honeypot.js     # 蜜罐端点
├── db/
│   ├── init.js         # Schema + 种子加载
│   └── seed.json       # AES 加密种子数据（运行时解密）
└── lib/
    └── logger.js       # 结构化审计日志

scripts/
├── gen-seed.js         # 种子数据 AES 加密工具
└── smoke-test.js       # 冒烟测试脚本
```

---

## 故障排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `node:sqlite module not found` | Node 版本 < 22 | 升级 Node |
| `EADDRINUSE` | 端口被占用 | `PORT=xxxx node src/server.js` |
| 429 Too Many Requests | 速率限制触发 | 降低频率或等冷却 |
| 403 Request blocked | WAF 触发 | 响应头 `X-Defense-Rule` 含规则名 |

---

© 2026 NovaPay Defense System · 由 **CatPaw 妙手 AI 助手**（美团）辅助构建
