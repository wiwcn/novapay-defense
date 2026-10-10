# NovaPay

一个模拟金融交易 HTTP 服务，作为网络安全攻防演练中的防守目标。

## 快速开始

### 环境要求
- Node.js ≥ 22（推荐 v22 LTS 及以上）
- 无 npm 依赖，系统零外部安装

### 解密与启动

```bash
# 解密（密钥由演练组织者提供）
node -e "
const crypto = require('crypto');
const fs = require('fs');
const data = fs.readFileSync('release.tar.gz.enc');
// 替换 <KEY> 为你收到的 32 字节密钥字符串
const key = crypto.createHash('sha256').update('<KEY>').digest();
const iv = data.slice(0, 16);
const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
fs.writeFileSync('release.tar.gz', Buffer.concat([decipher.update(data.slice(16)), decipher.final()]));
"
tar xzf release.tar.gz

# 启动服务
PORT=8080 node src/server.js
```

服务启动后输出：
```
[NovaPay] Server listening on 0.0.0.0:8080
[NovaPay] Ready for attack/defense testing.
```

## 文档

详细 API 接口、测试规则、故障排查见 [DEPLOY-GUIDE.md](./DEPLOY-GUIDE.md)。

## 文件结构

```
release.tar.gz.enc   ← 加密的源码包（AES-256-CBC）
DEPLOY-GUIDE.md      ← 部署与测试文档
README.md            ← 本文件
```

## 关联仓库

本作品由 **CatPaw 妙手 AI 助手**（美团）辅助构建。
