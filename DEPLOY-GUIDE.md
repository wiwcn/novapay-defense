# NovaPay Defense — 部署与测试手册（裁判/内部参考）

## 解密密钥

```
NOVAPAY-RELEASE.KEY
```

（已写入 README.md 供对手使用）

## 种子用户

| 用户名 | 密码 | 角色 | 余额 | is_honey |
|--------|------|------|------|----------|
| alice | password123 | user | 50000 | 0 |
| bob | b0bpass! | user | 30000 | 0 |
| admin | admin1234! | admin | 99999 | 0 |
| carol | carol2024! | user | 75000 | 0 |
| dave | davePw#88 | user | 12000 | 0 |
| sys_admin | *(fake)* | admin | 99999999 | 1 |
| root | *(fake)* | admin | 88888888 | 1 |
| backup_svc | *(fake)* | user | 77777777 | 1 |
| finance_api | *(fake)* | admin | 66666666 | 1 |

Honey 账号密码为假哈希，无法登录，但出现在 `/api/admin/users` 管理视图中。任何对其的查询 / 交易操作触发 `critical` 审计。

## 蜜罐端点

| 路径 | 内容 |
|------|------|
| `/.env` | 假 AWS key + 数据库密码 |
| `/.git/config` | 假 git 仓库地址 |
| `/admin.php` | 假 admin 登录表单 |
| `/wp-login.php` | 假 WP 登录表单 |
| `/phpmyadmin/` | 假 phpMyAdmin 界面 |
| `/api/debug` | 假调试信息 |

所有蜜罐触碰 → `X-Honey-Trap: triggered` 响应头 + critical 审计日志。

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| PORT | 8080 | HTTP 端口 |
| DB_PATH | :memory: | SQLite 路径（`:memory:` 为不持久化） |
| JWT_SECRET | default-insecure-secret | 生产必须设随机值 |
| ACTIVE_DEFENSE | false | true = 解锁 Phase 2 主动防御 |

## Phase 2 解锁

```bash
ACTIVE_DEFENSE=true JWT_SECRET=$(openssl rand -hex 32) PORT=8080 node src/server.js
```

解锁后能力：
- 行为匹配立即阻断（不仅告警）
- Tarpitting：对可疑 IP 的响应人为延迟 8 秒
- `/api/search` 投喂伪造高价值结果

## 架构摘要

```
每请求流水线（由外到内）:
1. Honeypot path    → 假端点返回诱饵 + 审计
2. WAF (path+query) → 路径/SQLi/XSS + 4 层 URL 解码
3. Body read        → 限 64KB
4. WAF (body)       → 同上规则扫描 JSON body
5. PromptGuard      → 零宽字符 + 同形字 + AI 越狱模式
6. Rate limit       → 滑动窗口: IP全局 200/60s, IP+path 20/60s, auth 5/60s
7. Behavior         → 请求序列匹配
8. JWT verify       → HS256 + 恒定时间签名校验
9. Handler          → 参数化 SQL 执行 (prepare + bind)
10. Audit log       → 结构化写入 defense_log
```

防守方保证：
- 所有 DB 操作使用参数化查询，无字符串拼接
- 所有密码 PBKDF2-SHA256 哈希存储，不存明文
- JWT 校验使用恒定时间比较，不泄漏时间侧信道
