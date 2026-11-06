# HomeGuard AI · 智能家居与家庭安全一体化平台

基于 **FastAPI + SQLite + MQTT + Vue 3 + ESP32** 的本地智能家居监控平台。ESP32 节点采集真实传感器数据，经 MQTT 上报到后端，在「智能设备」页面实时展示；漏水等事件会触发分级告警。

> 本仓库由本机实际运行的项目整理而来，包含平台源码、全部 ESP32 固件与部署文档。
> 项目原始说明见 [`docs/PROJECT_ORIGINAL.md`](docs/PROJECT_ORIGINAL.md)。

---

## ⚠️ 上传 / 使用前必读

**本仓库已对真实凭据做脱敏处理，克隆后不能直接烧录或运行，需要先填回配置：**

| 项目 | 占位符 | 需要处理的位置 |
|---|---|---|
| WiFi 名称 | `YOUR_WIFI_SSID` / `你的WiFi名` | 各固件 `WIFI_SSID` |
| WiFi 密码 | `YOUR_WIFI_PASSWORD` / `你的WiFi密码` | 各固件 `WIFI_PASSWORD` |
| MQTT Broker IP | `192.168.1.100` / `192.168.1.200` | 各固件 `MQTT_HOST` / `MQTT_HOSTS[]` |
| 本机用户名路径 | `C:\Users\<YOUR_USERNAME>` | `tools/flash/flash_leak.ps1` |
| 后端配置 | 复制 `backend/.env.example` 为 `.env` | `.env` 已被 `.gitignore` 排除 |

**安全提醒：**

- 后端默认管理员账号为 `admin / admin123`（项目自带演示值，见 `backend/app/config.py`）。**仅限本地演示，公开部署前务必修改，并启用鉴权。**
- 不要把 `.env`、`*.db`、编译产物 `*.bin`、flash 备份提交到仓库，`.gitignore` 已覆盖。
- 所有固件默认连接**局域网 MQTT Broker**，未做 TLS/认证，请只在可信内网使用。
- `docs/ARCHITECTURE.md` 中保留了开发用 ESP32 的 MAC 地址（用于区分板子），如介意可自行删除。

---

## 技术栈

| 层 | 技术 |
|---|---|
| 后端 | Python 3 + FastAPI + SQLAlchemy + Pydantic |
| 数据库 | SQLite |
| 消息通信 | MQTT（Mosquitto / `tools/jimi/broker.py`） |
| 前端 | Vue 3 + Vite + Element Plus + Pinia + Axios |
| 硬件 | ESP32 / ESP32-S3（Arduino 核心 3.3.12） |
| 固件依赖 | PubSubClient 2.8、ArduinoJson 7.x |

---

## 目录结构

```
HomeGuard-AI-GitHub/
├── backend/
│   ├── app/                  # FastAPI：main / config / models / schemas / services
│   │   ├── routers/          # auth / devices / overview / alarms / robot / scenes / ...
│   │   └── ai/               # 白名单工具 + LLM/规则兜底
│   ├── db/init.py            # 建表与种子设备
│   ├── requirements.txt
│   └── .env.example          # 配置模板（复制为 .env 使用）
├── frontend/
│   ├── src/views/            # 各页面，Devices.vue 为「智能设备」页
│   ├── src/api/              # axios + WebSocket
│   ├── package.json / vite.config.js
│   └── index.html
├── firmware/
│   ├── temp_humi_leak_node/  # ★ 当前在用：DHT11 + LM393 双节点共板
│   ├── common/               # TempHumiProbe.h（DHT/I2C 自动探测）
│   ├── env_monitor/          # 原环境监测中心固件（传感器抽象层）
│   ├── house_node/           # 原全屋节点
│   ├── led_status/  robot/   # 原配套固件
│   └── legacy/               # 历史迭代固件（MQ135 / MQ-2 / TGS2602 / PIR / ...）
├── tools/
│   ├── backend/              # 设备注册、改名、MQTT 抓包等维护脚本
│   ├── jimi/                 # broker / mock_device 等
│   ├── flash/                # 一键烧录脚本存档
│   └── start.bat             # 本机启动入口（路径需按本机调整）
├── config/                   # GPIO 映射、MQTT 主题、阈值说明
└── docs/
    ├── ARCHITECTURE.md       # 设备 / 引脚 / 数据流 / MQTT 协议
    ├── WIRING.md             # 各传感器接线
    ├── FLASHING.md           # 编译、烧录、校验、回退
    ├── API.md                # 接口说明
    └── PROJECT_ORIGINAL.md   # 项目原始 README
```

---

## 快速启动

### 1. MQTT Broker

```bash
docker run -d --name mosquitto -p 1883:1883 eclipse-mosquitto
# 或使用仓库自带的简易 broker：python tools/jimi/broker.py
```

### 2. 后端

```bash
cd backend
python -m venv .venv && .venv\Scripts\activate     # Windows
pip install -r requirements.txt
cp .env.example .env                              # 按需修改
python -m db.init                                 # 可选，启动时自动建表
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

接口文档：<http://127.0.0.1:8000/docs>

### 3. 前端

```bash
cd frontend
npm install
npm run dev
```

控制台：<http://127.0.0.1:5173/devices>（智能设备页）

> 默认 Vite 只监听 `127.0.0.1`。若需局域网访问，用 `npm run dev -- --host 0.0.0.0`。

### 4. 固件

1. 按 [`docs/WIRING.md`](docs/WIRING.md) 接线；
2. 填回 `WIFI_SSID` / `WIFI_PASSWORD` / `MQTT_HOST`；
3. 按 [`docs/FLASHING.md`](docs/FLASHING.md) 编译烧录；
4. 上电后节点自动连 WiFi/MQTT，卡片变为在线。

---

## 当前硬件状态

| 设备 | 传感器 | 固件 | 关键引脚 |
|---|---|---|---|
| `temp_living` 客厅温湿度 | DHT11 | `temp_humi_leak_node` | DATA → GPIO22 |
| `leak_kitchen` 厨房漏水 | LM393 水浸 | `temp_humi_leak_node`（同一块板） | DO → GPIO17，AO → GPIO34 |
| `smart_smoke_01` 烟雾 | MQ-2 | `legacy/mq2_node` | AO → GPIO34 |
| `air_quality_kitchen` 空气质量 | TGS2602 | `legacy/tgs2602_node` | AO → GPIO35 |
| `pir_living` 人体感应 | HC-SR501 | `legacy/pir_node`（ESP32-S3） | OUT → 自动扫描 |

**当前实际部署**：DHT11 与 LM393 接在**同一块 ESP32** 上，由 `temp_humi_leak_node` 一份固件同时上报两个设备，互不覆盖。其余节点固件保留在 `firmware/legacy/` 作为可回退版本，给对应板子供电即可自动重连。

完整的设备 ID / MAC / 报文字段见 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

---

## 数据链路

```
ESP32 传感器 → homeguard/house/status → MQTT Broker
     → backend/app/mqtt_client.py（订阅 homeguard/#）
     → services.handle_house_status() → upsert_device_state()
     → SQLite DeviceState.state（JSON）
     → GET /api/devices → frontend/src/views/Devices.vue → deviceDataText() 卡片显示
```

事件（如 `water_leak`）走 `homeguard/house/event`，由 `alarm_engine.py` 判为 `critical` 并生成告警。

---

## 文档

| 文档 | 内容 |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | 设备清单、板子 MAC、引脚分配、MQTT 主题与字段、后端处理链路 |
| [`docs/WIRING.md`](docs/WIRING.md) | 每种传感器的接线与电气注意事项 |
| [`docs/FLASHING.md`](docs/FLASHING.md) | 环境、编译、四分区烧录、校验、备份回退、常见坑 |
| [`docs/API.md`](docs/API.md) | 后端接口说明 |
| [`firmware/README.md`](firmware/README.md) | 全部固件索引与版本说明 |

---

## 安全边界

- AI 不得直接生成任意 MQTT 主题 / GPIO / 电机 PWM，只能通过 `backend/app/security.py` 的白名单动作调用。
- **停止命令最高优先级**，可打断巡逻和普通指令。
- 本地告警独立于网络/AI，断网仍可工作。
- 密码使用 PBKDF2-HMAC-SHA256 加盐哈希，含登录限流、账号锁定、会话管理与 CSRF token。

---

## License

未指定。上传 GitHub 前请在仓库设置中选择许可证；若用于公开项目，建议补充 `LICENSE` 文件。
