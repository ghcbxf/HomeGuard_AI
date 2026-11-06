# HomeGuard AI — 云端检测平台（Web 端）

基于 **FastAPI + SQLite + MQTT + Vue 3** 的家庭安全/智能家居监控平台。
参考《开发计划与实施路线图》与《一体化系统方案》两份文档实现。
<img width="1527" height="913" alt="{1DC3039C-6E43-4915-A26A-F2CAA35C87EA}" src="https://github.com/user-attachments/assets/a1ce87c5-ac42-482f-a599-998e19b41214" />

<img width="1859" height="946" alt="{4ABD525A-5B83-423F-A00A-5D325F71116E}" src="https://github.com/user-attachments/assets/d235b911-e186-45d5-b12d-3ea973869c70" />



## 技术栈

| 层 | 技术 |
|---|---|
| 后端 | Python + FastAPI + SQLAlchemy |
| 数据库 | SQLite |
| 消息通信 | MQTT（Mosquitto） |
| 前端 | Vue 3 + Vite + Element Plus + Pinia |
| AI 语音 | OpenAI 兼容 LLM（可选）+ 白名单工具 + 规则兜底 |

## 目录结构

```
HomeGuard_AI/
├── backend/          # FastAPI 后端
│   ├── app/
│   │   ├── main.py         # 入口
│   │   ├── config.py       # 配置（pydantic-settings）
│   │   ├── models.py       # 数据模型
│   │   ├── schemas.py      # 请求/响应模型
│   │   ├── services.py     # 业务层（MQTT 入库）
│   │   ├── security.py     # 白名单动作 + 参数校验 + 鉴权
│   │   ├── mqtt_client.py  # MQTT 客户端
│   │   ├── routers/        # auth / devices / robot / scenes / alarms / environment / voice
│   │   └── ai/             # tools.py（白名单工具）+ agent.py（LLM/兜底）
│   ├── db/init.py          # 建表脚本
│   └── requirements.txt
├── frontend/         # Vue 3 前端
│   └── src/
│       ├── views/          # 登录/总览/设备/机器人/... 等视图
│       ├── stores/         # Pinia（auth 登录态）
│       ├── api/            # axios 封装 + WebSocket 客户端
│       ├── router.js       # 含路由守卫
│       └── App.vue
├── firmware/         # ESP32 固件（占位说明）
├── config/           # GPIO 映射、MQTT 主题、阈值
└── docs/             # 文档
```

## 快速启动

### 1. 后端

```bash
cd backend
pip install -r requirements.txt

# 初始化数据库（可选，启动时也会自动建表）
python -m db.init

# 启动（需先有 MQTT Broker，见下）
uvicorn app.main:app --reload --port 8000
```

访问 API 文档：http://127.0.0.1:8000/docs

### 2. MQTT Broker（Mosquitto）

```bash
# Docker 方式（推荐）
docker run -d --name mosquitto -p 1883:1883 eclipse-mosquitto

# 或本机安装 mosquitto 后直接启动
mosquitto -c mosquitto.conf
```

### 3. 前端

```bash
cd frontend
npm install
npm run dev
```

访问控制台：http://127.0.0.1:5173

### 默认账号

首次启动自动创建管理员账号：**admin / admin123**（登录后可自行修改）。

## 配置

复制 `backend/.env.example` 为 `backend/.env`，按需修改：
- MQTT 地址/账号
- AI 语音（`HOMEGUARD_LLM_API_BASE` / `HOMEGUARD_LLM_API_KEY`）
- 会话与限流（`HOMEGUARD_SESSION_TTL_SECONDS` / `HOMEGUARD_MAX_LOGIN_ATTEMPTS` / `HOMEGUARD_LOCKOUT_SECONDS` 等）

## 安全边界（重要）

- **AI 不得直接生成任意 MQTT 主题 / GPIO / 电机 PWM**，只能通过白名单工具调用。
- 白名单动作定义在 `backend/app/security.py` 的 `ALLOWED_ACTIONS`。
- **停止命令最高优先级**，可打断巡逻和普通指令。
- **本地告警独立于网络/AI**（固件端实现），断网仍工作。
- 首版仅本地网络；远程访问需加认证 + 加密 + 访问控制。

## 认证与安全

- **密码存储**：PBKDF2-HMAC-SHA256 加盐哈希（`security.py` 的 `hash_password`）。
- **登录限流**：同一 IP 60 秒内最多 10 次尝试（`rate_limit_max_attempts`）。
- **账号锁定**：连续失败 5 次锁定 15 分钟（`max_login_attempts` / `lockout_seconds`）。
- **会话管理**：随机令牌 + 7 天有效期 + 2 小时空闲超时，支持主动下线。
- **CSRF**：登录返回 CSRF token，前端可在敏感操作中携带。
- **设备管理**：`/api/auth/devices` 查看已登录设备（名称/IP/时间），`/api/auth/devices/{id}/revoke` 强制下线。
