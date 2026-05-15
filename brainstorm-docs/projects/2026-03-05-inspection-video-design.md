# 巡课视频实时预览功能设计文档

## 一、需求概述

在现有设备管理列表页面，为每个设备增加"巡视"功能。点击"巡视"按钮后，弹出一个 Modal 窗口，展示该设备的实时视频流。

## 二、技术方案

### 2.1 视频流架构

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   Rust客户端    │       │   服务端        │       │   前端页面      │
│  (教室设备)     │       │  (Java后端)    │       │  (Vue浏览器)   │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ 1. 采集摄像头  │       │                 │       │                 │
│ 2. 编码为JPEG  │       │                 │       │                 │
│ 3. Base64编码  │──────▶│  视频服务       │──────▶│  FLV.js播放    │
│ 4. 推送视频帧  │       │  (被动拉流)     │       │  (Modal弹窗)   │
└─────────────────┘       └─────────────────┘       └─────────────────┘
```

### 2.2 视频传输方式

- **协议**：FLV over HTTP
- **延迟**：3-10秒（满足需求）
- **推送频率**：客户端每秒推送1-5帧
- **服务端缓存**：内存缓存最新视频帧

## 三、接口设计

### 3.1 客户端推送视频帧

**接口**：`POST /client/inspection/video/push`

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| deviceCode | String | 是 | 设备编码 |
| data | String | 是 | Base64编码的JPEG图片 |

**返回**：
```json
{
  "code": 200,
  "data": { "success": true },
  "msg": "success"
}
```

### 3.2 前端获取视频流地址

**接口**：`GET /client/inspection/video/url?deviceCode={deviceCode}`

**返回**：
```json
{
  "code": 200,
  "data": {
    "streamUrl": "/client/inspection/video/stream/DEV_001",
    "status": "online",
    "lastPushTime": "2026-03-05T10:30:00"
  },
  "msg": "success"
}
```

### 3.3 获取FLV视频流

**接口**：`GET /client/inspection/video/stream/{deviceCode}`

**返回**：FLV视频流（Content-Type: video/x-flv）

## 四、后端模块设计

### 4.1 新增文件

```
yudao-module-erp/src/main/java/cn/iocoder/yudao/module/erp/
├── controller/client/inspection/
│   └── ErpInspectionVideoController.java    # 视频接口控制器
├── service/inspection/
│   ├── ErpInspectionVideoService.java         # 视频服务接口
│   └── ErpInspectionVideoServiceImpl.java    # 视频服务实现
├── dal/dataobject/inspection/
│   └── ErpInspectionVideoFrameDO.java        # 视频帧缓存对象
└── enums/
    └── InspectionVideoStatusEnum.java        # 视频状态枚举
```

### 4.2 视频缓存策略

- 使用 ConcurrentHashMap 存储设备最新视频帧
- 缓存key：deviceCode
- 缓存value：byte[]（原始图片数据）
- 过期时间：30秒（超过30秒无新帧视为离线）
- 定期清理：每10秒检查一次

### 4.3 核心类设计

**ErpInspectionVideoServiceImpl**：
```java
@Service
public class ErpInspectionVideoServiceImpl implements ErpInspectionVideoService {

    // 视频帧缓存：deviceCode -> byte[]
    private final ConcurrentHashMap<String, VideoFrame> frameCache = new ConcurrentHashMap<>();

    @Override
    public void pushFrame(String deviceCode, byte[] frameData) {
        // 1. 验证设备是否存在
        // 2. 更新缓存
        frameCache.put(deviceCode, new VideoFrame(frameData, System.currentTimeMillis()));
    }

    @Override
    public VideoStreamInfo getStreamInfo(String deviceCode) {
        // 1. 获取缓存
        // 2. 检查是否过期
        // 3. 返回流信息
    }

    @Override
    public void writeStream(HttpServletResponse response, String deviceCode) {
        // 1. 设置响应头
        // 2. 持续输出FLV流（模拟FLV封装）
    }
}
```

## 五、前端模块设计

### 5.1 修改文件

- `src/views/hc/inspection/device/index.vue` - 添加"巡视"按钮
- `src/views/hc/inspection/device/VideoInspectModal.vue` - 新增视频弹窗组件

### 5.2 视频弹窗组件设计

```vue
<template>
  <Dialog v-model="visible" title="设备巡视" width="800px">
    <div class="video-container">
      <video ref="videoRef" class="video-player" />
      <div class="video-info">
        <span>设备：{{ deviceName }}</span>
        <span>状态：{{ status }}</span>
      </div>
    </div>
  </Dialog>
</template>

<script setup>
import flvjs from 'flv.js'

// 使用 flv.js 播放FLV流
// 定时刷新流地址
</script>
```

### 5.3 依赖

需要引入 flv.js：
```bash
pnpm add flv.js
```

## 六、数据表设计

### 6.1 视频记录表（可选）

如果需要记录巡视历史，可创建：

```sql
CREATE TABLE erp_inspection_video_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    device_id BIGINT NOT NULL COMMENT '设备ID',
    device_code VARCHAR(64) NOT NULL COMMENT '设备编码',
    user_id BIGINT COMMENT '巡视人ID',
    user_name VARCHAR(64) COMMENT '巡视人姓名',
    start_time DATETIME NOT NULL COMMENT '开始时间',
    end_time DATETIME COMMENT '结束时间',
    create_time DATETIME,
    update_time DATETIME,
    deleted BIT DEFAULT FALSE
);
```

**说明**：当前版本先不实现历史记录，仅实现实时预览。

## 七、实施步骤

### 第一步：后端 - 视频服务
1. 创建 ErpInspectionVideoController
2. 实现 pushFrame 接口（接收视频帧）
3. 实现 getStreamInfo 接口（获取流信息）
4. 实现 getStream 接口（输出FLV流）

### 第二步：后端 - 设备服务改造
1. ErpInspectionDeviceService 增加获取设备状态方法

### 第三步：前端 - 巡视功能
1. 安装 flv.js 依赖
2. 设备列表添加"巡视"按钮
3. 创建 VideoInspectModal 组件
4. 实现 FLV 视频播放

### 第四步：测试
1. 测试视频帧推送接口
2. 测试FLV流播放
3. 测试多设备同时在线

## 八、验收标准

1. 客户端能够成功推送视频帧到服务端
2. 前端点击"巡视"按钮能够打开视频弹窗
3. 视频弹窗能够播放实时视频流
4. 视频延迟在3-10秒以内
5. 设备离线时能够正确显示状态

## 九、风险与限制

1. **当前方案为简化实现**：服务端直接输出缓存的图片为FLV格式，延迟约3-5秒
2. **未使用专业流媒体服务器**：如需更低延迟可后续集成SRS/nginx-rtmp
3. **仅支持JPEG格式**：客户端需推送JPEG图片
4. **单设备测试**：初期先测试单设备，后续扩展多设备
