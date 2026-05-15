# 远程电子巡课功能设计

> 创建日期：2026-03-04
> 作者：Claude Code
> 状态：设计完成，待开发

## 1. 需求概述

### 1.1 功能描述

远程电子巡课功能通过安装在智能黑板/智能多媒体设备上的客户端，远程调用摄像头查看当前教室画面，实现教学场景的远程巡查和监控。

### 1.2 需求总结

| 维度 | 要求 |
|------|------|
| 目标模块 | ERP 模块（ErpInspection） |
| 视频功能 | 实时视频流 + 按需截图 |
| 设备接入 | 主动注册 + 手动添加 |
| 日志记录 | 谁在何时看了哪个教室 |
| 客户端 | Rust 桌面客户端（已实现截图） |

## 2. 整体架构

### 2.1 系统架构

采用前后端分离 + 后端转发架构：

**后端扩展**（`yudao-module-erp`）：
- 新增 `inspection` 子模块
- 新增数据库表：`erp_inspection_device`（设备管理）、`erp_inspection_log`（查看日志）
- 设备注册/心跳服务
- 视频流转发服务（截图/API）

**前端扩展**（`front/hc_vue`）：
- 管理端：设备管理页面
- 巡课端：视频查看页面

**客户端**（Rust）：
- 截图客户端（已有）
- 后续扩展实时视频推流

### 2.2 数据流向

```
[智能设备/Rust客户端]  ──注册/心跳──>  [后端服务]
                                               │
                                               ▼
[前端浏览器] <───视频/截图URL──  [后端转发服务]
     │
     ▼
[查看日志入库]
```

### 2.3 技术方案

| 功能 | 技术方案 |
|------|----------|
| 设备通信 | HTTP REST API + WebSocket 心跳 |
| 截图获取 | 客户端主动上报/后端拉取 |
| 实时视频 | WebRTC 或 RTMP（后续扩展） |
| 日志存储 | MySQL 数据库 |

## 3. 数据库设计

### 3.1 新增表：erp_inspection_device

```sql
CREATE TABLE `erp_inspection_device` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `device_name` varchar(100) NOT NULL COMMENT '设备名称',
  `device_code` varchar(64) NOT NULL COMMENT '设备编码（唯一标识）',
  `device_type` tinyint NOT NULL DEFAULT '1' COMMENT '设备类型：1-智能黑板，2-智能多媒体，3-其他',
  `classroom_id` bigint DEFAULT NULL COMMENT '关联教室ID（可选）',
  `classroom_name` varchar(100) DEFAULT NULL COMMENT '教室名称',
  `ip_address` varchar(50) DEFAULT NULL COMMENT '设备IP地址',
  `port` int DEFAULT NULL COMMENT '设备端口',
  `status` tinyint NOT NULL DEFAULT '1' COMMENT '在线状态：1-在线，0-离线',
  `register_type` tinyint NOT NULL DEFAULT '1' COMMENT '注册方式：1-主动注册，2-手动添加',
  `last_heartbeat` datetime DEFAULT NULL COMMENT '最后心跳时间',
  `last_screenshot_time` datetime DEFAULT NULL COMMENT '最后截图时间',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  `creator` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_device_code` (`device_code`),
  KEY `idx_status` (`status`),
  KEY `idx_classroom_id` (`classroom_id`)
) ENGINE=InnoDB COMMENT='巡课设备表';
```

### 3.2 新增表：erp_inspection_log

```sql
CREATE TABLE `erp_inspection_log` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `device_id` bigint NOT NULL COMMENT '设备ID',
  `device_name` varchar(100) NOT NULL COMMENT '设备名称',
  `classroom_name` varchar(100) DEFAULT NULL COMMENT '教室名称',
  `user_id` bigint NOT NULL COMMENT '查看人ID',
  `user_name` varchar(64) NOT NULL COMMENT '查看人姓名',
  `view_type` tinyint NOT NULL COMMENT '查看类型：1-实时视频，2-截图查看',
  `view_start_time` datetime NOT NULL COMMENT '查看开始时间',
  `view_end_time` datetime DEFAULT NULL COMMENT '查看结束时间',
  `ip_address` varchar(50) DEFAULT NULL COMMENT '查看人IP',
  `creator` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`),
  KEY `idx_device_id` (`device_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_view_start_time` (`view_start_time`)
) ENGINE=InnoDB COMMENT='巡课查看日志表';
```

## 4. 后端 API 设计

### 4.1 设备端 API（客户端调用）

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 设备注册 | POST | `/client-api/erp/inspection/device/register` | 客户端主动注册 |
| 心跳上报 | POST | `/client-api/erp/inspection/device/heartbeat` | 客户端定时心跳 |
| 上报截图 | POST | `/client-api/erp/inspection/screenshot/upload` | 客户端上传截图 |
| 获取截图 | GET | `/client-api/erp/inspection/screenshot/{deviceCode}` | 后端拉取最新截图 |

### 4.2 管理端 API（后台管理）

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 设备列表 | GET | `/admin-api/erp/inspection/device/page` | 分页查询设备 |
| 设备详情 | GET | `/admin-api/erp/inspection/device/{id}` | 获取设备详情 |
| 新增设备 | POST | `/admin-api/erp/inspection/device` | 手动添加设备 |
| 更新设备 | PUT | `/admin-api/erp/inspection/device/{id}` | 更新设备信息 |
| 删除设备 | DELETE | `/admin-api/erp/inspection/device/{id}` | 删除设备 |
| 查看日志 | GET | `/admin-api/erp/inspection/log/page` | 分页查询查看日志 |

### 4.3 巡课端 API（前端调用）

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取设备列表 | GET | `/api/erp/inspection/device/online` | 获取在线设备列表 |
| 获取实时视频流 | GET | `/api/erp/inspection/stream/{deviceCode}` | 获取实时视频流地址 |
| 获取最新截图 | GET | `/api/erp/inspection/screenshot/{deviceCode}` | 获取最新截图 |
| 开始查看 | POST | `/api/erp/inspection/view/start` | 记录开始查看 |
| 结束查看 | POST | `/api/erp/inspection/view/end` | 记录结束查看 |

### 4.4 响应示例

**设备列表响应：**
```json
{
  "list": [
    {
      "id": 1,
      "deviceName": "一年级一班黑板",
      "deviceCode": "DEVICE_001",
      "deviceType": 1,
      "classroomName": "一年级一班",
      "status": 1,
      "registerType": 1,
      "lastHeartbeat": "2026-03-04 10:30:00",
      "lastScreenshotTime": "2026-03-04 10:28:00"
    }
  ],
  "total": 10
}
```

**截图响应：**
```json
{
  "deviceCode": "DEVICE_001",
  "screenshotUrl": "https://xxx.oss.com/screenshots/DEVICE_001.jpg",
  "screenshotTime": "2026-03-04 10:28:00"
}
```

## 5. 前端设计

### 5.1 管理端页面

**设备管理页面：**
- 设备列表（表格）
- 新增设备（手动添加）
- 编辑/删除设备
- 设备详情（查看在线状态、心跳时间）
- 查看日志入口

**日志查询页面：**
- 分页列表（设备名称、查看人、时间范围）
- 导出功能

### 5.2 巡课页面

**巡课主页：**
- 教室/设备列表（网格或列表视图）
- 在线状态指示
- 支持搜索/筛选

**视频查看页面：**
- 实时视频播放器
- 截图查看区域
- 刷新按钮
- 查看日志自动记录

## 6. 核心实现

### 6.1 设备注册流程

```
1. 客户端启动，向后端发送注册请求
2. 后端验证设备编码（可配置白名单）
3. 首次注册：创建设备记录
4. 已存在：更新设备信息
5. 返回注册成功响应（含设备ID）
6. 客户端开始定时心跳
```

### 6.2 心跳机制

```java
// 心跳间隔：30秒
// 离线判定：连续3次心跳超时（约90秒）

public class DeviceHeartbeatService {
    public void handleHeartbeat(HeartbeatRequest request) {
        // 1. 查找设备
        InspectionDeviceDO device = deviceMapper.selectByCode(request.getDeviceCode());

        // 2. 更新心跳时间
        device.setLastHeartbeat(new Date());
        device.setStatus(DeviceStatusEnum.ONLINE.getStatus());
        deviceMapper.updateById(device);
    }
}
```

### 6.3 查看日志记录

```java
public class InspectionViewService {
    public Long startView(Long deviceId, Long userId) {
        // 1. 创建查看记录
        InspectionLogDO log = new InspectionLogDO();
        log.setDeviceId(deviceId);
        log.setUserId(userId);
        log.setViewStartTime(new Date());
        log.setViewType(ViewTypeEnum.REAL_TIME.getType());
        logMapper.insert(log);

        // 2. 返回日志ID（用于结束记录）
        return log.getId();
    }

    public void endView(Long logId) {
        InspectionLogDO log = logMapper.selectById(logId);
        log.setViewEndTime(new Date());
        logMapper.updateById(log);
    }
}
```

### 6.4 截图存储策略

| 存储方式 | 说明 |
|----------|------|
| OSS/OSS | 截图上传到对象存储，返回URL |
| 本地存储 | 仅开发测试环境使用 |
| 定时清理 | 保留最近24小时的截图 |

## 7. 错误处理

### 7.1 异常场景

| 场景 | 处理方式 |
|------|----------|
| 设备离线 | 显示离线状态，前端提示"设备已离线" |
| 截图获取失败 | 显示占位图，提示"截图获取失败" |
| 心跳超时 | 自动更新设备状态为离线 |
| 客户端未注册 | 拒绝请求，返回错误码 |
| 查看超时 | 自动结束查看记录 |

### 7.2 安全措施

- 设备编码需在后端白名单注册（可配置）
- 接口签名验证（可选）
- 登录用户才能查看视频
- 敏感操作日志审计

## 8. 测试策略

### 8.1 测试用例

- 设备注册/注销流程测试
- 心跳保活测试
- 截图上传/拉取测试
- 查看日志记录测试
- 设备状态同步测试
- 并发查看测试

### 8.2 测试范围

- 单设备单用户
- 单设备多用户
- 设备频繁上下线
- 网络异常场景

## 9. 部署方案

### 9.1 部署结构

```
后端：yudao-module-erp
  - Controller:
    - ErpInspectionDeviceClientController（客户端API）
    - ErpInspectionDeviceAdminController（管理API）
    - ErpInspectionStreamController（视频流API）
  - Service:
    - ErpInspectionDeviceService
    - ErpInspectionStreamService
    - ErpInspectionLogService
  - DAL:
    - ErpInspectionDeviceMapper
    - ErpInspectionLogMapper
  - DO:
    - ErpInspectionDeviceDO
    - ErpInspectionLogDO

前端：front/hc_vue
  - views/hc/inspection/device/（设备管理）
  - views/hc/inspection/log/（日志查看）
  - views/hc/inspection/view/（巡课查看）
```

### 9.2 目录结构

```
yudao-module-erp/src/main/java/cn/iocoder/yudao/module/erp/
├── controller/
│   ├── client/inspection/          # 客户端API
│   └── admin/inspection/          # 管理端API
├── service/
│   └── inspection/               # 巡课服务
├── dal/
│   ├── dataobject/inspection/    # 数据对象
│   └── mysql/inspection/         # 数据访问
└── ...
```

## 10. 实施计划

| 阶段 | 任务 | 时间 |
|------|------|------|
| 1 | 数据库设计与创建 | 0.5天 |
| 2 | 设备管理后端开发 | 1.5天 |
| 3 | 截图服务后端开发 | 1天 |
| 4 | 查看日志后端开发 | 1天 |
| 5 | 管理端前端开发 | 1.5天 |
| 6 | 巡课前端开发 | 1.5天 |
| 7 | 客户端对接开发 | 1天 |
| 8 | 联调测试 | 1.5天 |
| 9 | 部署上线 | 0.5天 |
| **合计** | | **10天** |

## 11. 后续扩展

- 实时视频流（WebRTC/RTMP）
- 视频录制与回放
- 设备批量管理
- 巡课报表统计
- 移动端巡课 App
- 与现有 labinfo 模块关联
