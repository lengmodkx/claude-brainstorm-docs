# 软件使用监控功能设计文档

## 功能概述

在客户端运行期间，记录用户打开的软件名称、运行时间，并将其推送到系统中。同时服务于教学监管、资产管理和行为分析。

## 架构设计

### 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│                         采集层 (Rust)                        │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ ProcessMonitor │  │ Windows API  │  │  SessionManager     │ │
│  │   (轮询)     │  │  (进程信息)   │  │  (会话管理)          │ │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬──────────┘ │
└─────────┼────────────────┼────────────────────┼────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                        存储层                               │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │   内存缓存   │  │   SQLite    │  │      服务端          │ │
│  │(ActiveSession)│  │(本地持久化)  │  │    (汇总分析)        │ │
│  └─────────────┘  └──────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                       同步层 (智能上报)                       │
│  ┌─────────────────┐        ┌──────────────────────────┐   │
│  │   实时通道       │        │        批量通道           │   │
│  │ (启动/关闭事件)  │        │   (5分钟累积上报)         │   │
│  └─────────────────┘        └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 数据模型

### 核心数据结构

```rust
// 软件使用会话记录
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SoftwareSession {
    pub id: String,              // UUID
    pub process_name: String,    // 进程名（如 notepad.exe）
    pub window_title: String,    // 窗口标题
    pub exe_path: String,        // 可执行文件路径
    pub start_time: i64,         // 启动时间戳（毫秒）
    pub end_time: Option<i64>,   // 关闭时间戳（毫秒）
    pub duration_secs: i64,      // 使用时长（秒）
    pub device_id: i64,          // 设备ID
}

// 上报数据结构
#[derive(Debug, Serialize)]
pub struct SoftwareUsageBatch {
    pub device_code: String,
    pub sessions: Vec<SoftwareSession>,
}

// 内存中活跃会话
pub struct ActiveSession {
    pub session: SoftwareSession,
    pub last_active_time: Instant,
}
```

### SQLite 表结构

```sql
-- 软件使用会话表
CREATE TABLE software_sessions (
    id TEXT PRIMARY KEY,
    process_name TEXT NOT NULL,
    window_title TEXT,
    exe_path TEXT,
    start_time INTEGER NOT NULL,
    end_time INTEGER,
    duration_secs INTEGER DEFAULT 0,
    device_id INTEGER NOT NULL,
    synced INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 同步队列表（断网续传）
CREATE TABLE sync_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    retry_count INTEGER DEFAULT 0,
    next_retry_at INTEGER,
    FOREIGN KEY (session_id) REFERENCES software_sessions(id)
);

-- 索引
CREATE INDEX idx_sessions_device_time ON software_sessions(device_id, start_time);
CREATE INDEX idx_sessions_synced ON software_sessions(synced);
```

## 组件设计

### 1. ProcessMonitor 模块

```rust
pub struct ProcessMonitor {
    active_session: Option<ActiveSession>,
    check_interval: Duration,
    config: MonitorConfig,
}

pub enum MonitorEvent {
    SessionStarted(SoftwareSession),
    SessionSwitched { from: String, to: SoftwareSession },
    SessionEnded(String), // session_id
    UsageUpdated { session_id: String, duration_secs: i64 },
}

impl ProcessMonitor {
    /// 定时轮询，生成事件
    pub fn tick(&mut self) -> Vec<MonitorEvent> {
        let current = self.get_current_process();
        // 对比活跃会话，生成相应事件
    }

    /// 获取当前前台窗口进程信息
    fn get_current_process(&self) -> Option<ProcessInfo> {
        // Windows API 调用
    }
}
```

### 2. Windows API 封装模块

```rust
pub struct ProcessInfo {
    pub pid: u32,
    pub name: String,
    pub exe_path: String,
    pub window_title: String,
}

/// 获取前台窗口信息
pub fn get_foreground_process() -> Result<ProcessInfo, String>;

/// 判断是否为系统进程
pub fn is_system_process(exe_path: &str) -> bool;

/// 获取所有用户进程列表
pub fn enumerate_user_processes() -> Vec<ProcessInfo>;
```

### 3. SessionManager 模块

```rust
pub struct SessionManager {
    db: Connection,
    active_session: Option<ActiveSession>,
}

impl SessionManager {
    /// 处理监控事件
    pub fn handle_event(&mut self, event: MonitorEvent) -> Result<(), String>;

    /// 结束当前会话并持久化
    pub fn close_active_session(&mut self) -> Result<(), String>;

    /// 查询待同步记录
    pub fn get_pending_sync(&self, limit: usize) -> Vec<SoftwareSession>;

    /// 标记记录已同步
    pub fn mark_synced(&mut self, session_ids: &[String]) -> Result<(), String>;
}
```

### 4. 同步调度器

```rust
pub struct SyncScheduler {
    batch_interval: Duration,
    last_batch_time: Instant,
}

pub enum SyncStrategy {
    Realtime,   // 立即上报（启动/关闭事件）
    Batch,      // 批量上报（累积数据）
}

impl SyncScheduler {
    /// 发送实时事件
    pub async fn send_realtime(&self, event: &MonitorEvent) -> Result<(), SyncError>;

    /// 执行批量同步
    pub async fn sync_batch(&self, sessions: &[SoftwareSession]) -> Result<(), SyncError>;
}
```

## 配置项

在 `AppConfig` 中新增：

```rust
pub struct AppConfig {
    // ... 现有字段 ...

    // 软件监控配置
    pub software_monitor_enabled: bool,      // 默认: true
    pub software_monitor_interval_secs: u32, // 轮询间隔，默认: 5秒
    pub software_monitor_batch_secs: u32,    // 批量上报间隔，默认: 300秒（5分钟）
    pub software_monitor_idle_threshold_secs: u32, // 闲置阈值，默认: 300秒（5分钟）
}
```

## 数据流

### 1. 采集循环（每5秒）

```
WinAPI获取前台窗口
    │
    ▼
过滤系统进程（可配置）
    │
    ▼
对比上一个会话
    │
    ├── 新进程 ──> 生成 Start 事件
    │
    ├── 切换进程 ──> 生成 Switch 事件（关闭旧 + 启动新）
    │
    ├── 同一进程 ──> 更新时长
    │
    └── 无进程 ──> 生成 End 事件（如果有活跃会话）
```

### 2. 实时上报通道

```
Start/End/Switch 事件
    │
    ▼
立即 HTTP POST /api/software/usage/realtime
    │
    ├── 成功 ──> 完成
    │
    └── 失败 ──> 加入 sync_queue，定时重试
```

### 3. 批量上报通道（每5分钟）

```
定时触发
    │
    ▼
查询未同步的使用时长记录
    │
    ▼
HTTP POST /api/software/usage/batch
    │
    ├── 成功 ──> 标记 synced=1，清空队列
    │
    └── 失败 ──> 记录下次重试时间
```

## 错误处理策略

| 场景 | 处理方式 |
|------|----------|
| 网络断开 | 数据持久化到 SQLite，网络恢复后从 sync_queue 重试 |
| 服务端5xx | 指数退避重试（1s → 2s → 4s → 8s → 16s），最多5次 |
| SQLite失败 | 降级到内存缓存，记录错误日志，避免数据丢失 |
| 进程信息获取失败 | 跳过该次轮询，记录警告日志，下次继续 |
| 服务端4xx | 标记为无效数据，不再重试，记录错误日志 |

## 闲置检测

- 窗口标题5分钟无变化 → 标记为闲置状态
- 闲置期间不计入使用时长
- 会话保持打开，直到窗口切换或关闭

## API 接口

### 实时上报接口

```http
POST /client/software/usage/realtime
Authorization: Bearer {token}
Content-Type: application/json

{
    "deviceCode": "DEVICE_xxx",
    "event": "started",  // started | ended | switched
    "session": {
        "id": "uuid",
        "processName": "notepad.exe",
        "windowTitle": "无标题 - 记事本",
        "exePath": "C:\\Windows\\notepad.exe",
        "startTime": 1709876543210,
        "deviceId": 123
    }
}
```

### 批量上报接口

```http
POST /client/software/usage/batch
Authorization: Bearer {token}
Content-Type: application/json

{
    "deviceCode": "DEVICE_xxx",
    "sessions": [
        {
            "id": "uuid-1",
            "processName": "chrome.exe",
            "windowTitle": "百度一下 - Google Chrome",
            "exePath": "C:\\Program Files\\Google\\Chrome\\chrome.exe",
            "startTime": 1709876543210,
            "endTime": 1709876843210,
            "durationSecs": 300,
            "deviceId": 123
        }
    ]
}
```

## 新增依赖

在 `src-tauri/Cargo.toml` 中添加：

```toml
[dependencies]
# 现有依赖...
sysinfo = "0.30"          # 跨平台进程信息
rusqlite = { version = "0.31", features = ["bundled"] }  # SQLite
uuid = { version = "1.7", features = ["v4"] }  # UUID生成

[target.'cfg(windows)'.dependencies]
windows = { version = "0.52", features = [
    "Win32_System_Threading",
    "Win32_UI_WindowsAndMessaging",
    "Win32_Foundation",
] }
```

## 实现步骤

1. **添加依赖**：更新 Cargo.toml
2. **数据库初始化**：创建 SQLite 表和索引
3. **实现 Windows API 封装**：获取进程信息、过滤系统进程
4. **实现 ProcessMonitor**：轮询和事件生成
5. **实现 SessionManager**：会话管理和数据持久化
6. **实现 SyncScheduler**：上报调度和重试机制
7. **集成到 App**：在 lib.rs 中启动监控服务
8. **前端配置界面**：添加监控配置选项

## 安全与隐私

- 仅监控用户应用程序，默认排除系统进程
- 数据本地加密存储（SQLite 加密或应用层加密）
- 配置化开关，用户可选择关闭监控
- 上报数据仅包含必要字段，不包含敏感内容

## 验收标准

- [ ] 能准确检测软件启动、切换、关闭事件
- [ ] 记录包含进程名、窗口标题、路径、时长
- [ ] 实时事件（启动/关闭）3秒内上报到服务端
- [ ] 批量数据5分钟内完成同步
- [ ] 网络断开后数据不丢失，恢复后自动补传
- [ ] 服务端接口返回4xx/5xx时有适当重试策略
- [ ] 可通过配置关闭监控功能
- [ ] 系统进程被正确过滤

---

设计日期：2026-03-09
版本：v1.0
