# 软件监控服务端设计文档

## 功能概述

接收客户端推送的软件使用数据，存储到MySQL数据库，并提供管理端查询接口。

## 数据库表设计

### 1. 软件使用明细表（保留30天）

```sql
CREATE TABLE software_usage_detail (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '编号',
    session_id VARCHAR(64) UNIQUE COMMENT '客户端会话ID',
    device_id BIGINT NOT NULL COMMENT '设备ID',
    device_code VARCHAR(64) COMMENT '设备编码',
    process_name VARCHAR(128) NOT NULL COMMENT '进程名',
    window_title VARCHAR(512) COMMENT '窗口标题',
    exe_path VARCHAR(512) COMMENT '可执行路径',
    start_time DATETIME NOT NULL COMMENT '开始时间',
    end_time DATETIME COMMENT '结束时间',
    duration_secs INT DEFAULT 0 COMMENT '使用时长(秒)',
    dept_id BIGINT COMMENT '学校/部门ID',
    class_id BIGINT COMMENT '班级ID',
    tenant_id BIGINT COMMENT '租户id',
    deleted BIT DEFAULT b'0' COMMENT '是否删除',
    creator VARCHAR(64) COMMENT '创建者',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater VARCHAR(64) COMMENT '更新者',
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    INDEX idx_device_time (device_id, start_time),
    INDEX idx_process (process_name, start_time),
    INDEX idx_class_time (class_id, start_time),
    INDEX idx_dept_time (dept_id, start_time),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='软件使用明细表';
```

### 2. 软件使用日汇总表（永久保留）

```sql
CREATE TABLE software_usage_daily (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '编号',
    device_id BIGINT NOT NULL COMMENT '设备ID',
    process_name VARCHAR(128) NOT NULL COMMENT '进程名',
    usage_date DATE NOT NULL COMMENT '使用日期',
    total_duration_secs INT DEFAULT 0 COMMENT '总使用时长(秒)',
    open_count INT DEFAULT 0 COMMENT '打开次数',
    last_window_title VARCHAR(512) COMMENT '最后窗口标题',
    class_id BIGINT COMMENT '班级ID',
    dept_id BIGINT COMMENT '学校/部门ID',
    tenant_id BIGINT COMMENT '租户id',
    deleted BIT DEFAULT b'0' COMMENT '是否删除',
    creator VARCHAR(64) COMMENT '创建者',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater VARCHAR(64) COMMENT '更新者',
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    UNIQUE KEY uk_device_process_date (device_id, process_name, usage_date),
    INDEX idx_class_date (class_id, usage_date),
    INDEX idx_dept_date (dept_id, usage_date),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='软件使用日汇总表';
```

## API 接口设计

### 1. 客户端推送接口

#### 实时上报
```http
POST /client/software/usage/realtime
Authorization: Bearer {token}
Content-Type: application/json

{
  "deviceCode": "DEVICE_xxx",
  "event": "started",
  "session": {
    "id": "uuid",
    "processName": "chrome.exe",
    "windowTitle": "百度一下，你就知道",
    "exePath": "C:\\Program Files\\Google\\Chrome\\chrome.exe",
    "startTime": 1709876543210,
    "deviceId": 123
  }
}
```

#### 批量上报
```http
POST /client/software/usage/batch
Authorization: Bearer {token}
Content-Type: application/json

{
  "deviceCode": "DEVICE_xxx",
  "sessions": [
    {
      "id": "uuid-1",
      "processName": "wps.exe",
      "windowTitle": "数学课件.pptx",
      "exePath": "C:\\Program Files\\WPS\\wps.exe",
      "startTime": 1709876543210,
      "endTime": 1709876843210,
      "durationSecs": 300,
      "deviceId": 123
    }
  ]
}
```

### 2. 管理端查询接口

#### 2.1 获取软件使用明细列表
```http
GET /admin-api/hc/software-usage/page?deviceId=123&startTime=2026-03-01%2000:00:00&endTime=2026-03-09%2023:59:59&pageNo=1&pageSize=20
```

**响应：**
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "total": 156,
    "list": [{
      "id": 12345,
      "sessionId": "uuid-xxxx",
      "deviceId": 123,
      "deviceName": "教室大屏01",
      "processName": "chrome.exe",
      "windowTitle": "百度一下，你就知道",
      "exePath": "C:\\Program Files\\Google\\Chrome\\chrome.exe",
      "startTime": "2026-03-09 14:30:00",
      "endTime": "2026-03-09 14:35:20",
      "durationSecs": 320,
      "durationText": "5分20秒",
      "classId": 456,
      "className": "三年级一班"
    }]
  }
}
```

#### 2.2 按软件统计使用时长
```http
GET /admin-api/hc/software-usage/statistics/by-software?classId=456&startDate=2026-03-01&endDate=2026-03-09&topN=10
```

**响应：**
```json
{
  "code": 0,
  "data": {
    "totalDevices": 25,
    "totalDurationSecs": 452300,
    "softwareStats": [{
      "rank": 1,
      "processName": "chrome.exe",
      "processDisplayName": "Google Chrome",
      "category": "浏览器",
      "totalDurationSecs": 125600,
      "durationText": "34小时53分",
      "deviceCount": 25,
      "avgDurationSecs": 5024,
      "openCount": 156
    }]
  }
}
```

#### 2.3 获取班级软件使用概览
```http
GET /admin-api/hc/software-usage/class-overview?classId=456&date=2026-03-09
```

**响应：**
```json
{
  "code": 0,
  "data": {
    "classId": 456,
    "className": "三年级一班",
    "date": "2026-03-09",
    "deviceCount": 3,
    "onlineDeviceCount": 2,
    "activeDeviceCount": 2,
    "totalUsageHours": 18.5,
    "topSoftware": [],
    "deviceStatusList": [{
      "deviceId": 123,
      "deviceName": "大屏01",
      "online": true,
      "currentProcess": "wps.exe",
      "currentWindowTitle": "数学课件.pptx",
      "todayUsageSecs": 15600
    }]
  }
}
```

## 项目结构

```
src/main/java/com/xxx/hc/modules/software/
├── controller/
│   ├── SoftwareUsageController.java          # 管理端接口
│   └── ClientSoftwareUsageController.java    # 客户端推送接口
├── service/
│   ├── SoftwareUsageService.java
│   ├── SoftwareUsageServiceImpl.java
│   └── SoftwareUsagePushService.java
├── dal/
│   ├── dataobject/
│   │   ├── SoftwareUsageDetailDO.java
│   │   └── SoftwareUsageDailyDO.java
│   ├── mapper/
│   │   ├── SoftwareUsageDetailMapper.java
│   │   └── SoftwareUsageDailyMapper.java
│   └── vo/
│       ├── SoftwareUsageVO.java
│       ├── SoftwareStatVO.java
│       └── ClassSoftwareOverviewVO.java
├── convert/
│   └── SoftwareUsageConvert.java
└── dto/
    ├── SoftwareUsageRealtimeReqDTO.java
    ├── SoftwareUsageBatchReqDTO.java
    ├── SoftwareUsagePageReqDTO.java
    └── SoftwareStatReqDTO.java
```

## DTO/VO 定义

### DTO

```java
// SoftwareUsageRealtimeReqDTO.java
@Data
public class SoftwareUsageRealtimeReqDTO {
    private String deviceCode;
    private String event;
    private SessionPayload session;
}

@Data
public class SessionPayload {
    private String id;
    private String processName;
    private String windowTitle;
    private String exePath;
    private Long startTime;
    private Long endTime;
    private Long durationSecs;
    private Long deviceId;
}

// SoftwareUsageBatchReqDTO.java
@Data
public class SoftwareUsageBatchReqDTO {
    private String deviceCode;
    private List<SessionPayload> sessions;
}

// SoftwareUsagePageReqDTO.java
@Data
public class SoftwareUsagePageReqDTO extends PageParam {
    private Long deviceId;
    private Long classId;
    private String processName;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
}

// SoftwareStatReqDTO.java
@Data
public class SoftwareStatReqDTO {
    private Long classId;
    private Long deptId;
    private Long deviceId;
    private LocalDate startDate;
    private LocalDate endDate;
    private Integer topN = 10;
}
```

### VO

```java
// SoftwareUsageVO.java
@Data
public class SoftwareUsageVO {
    private Long id;
    private String sessionId;
    private Long deviceId;
    private String deviceName;
    private String processName;
    private String windowTitle;
    private String exePath;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private Integer durationSecs;
    private String durationText;
    private Long classId;
    private String className;
    private String classroomName;
}

// SoftwareStatVO.java
@Data
public class SoftwareStatVO {
    private Integer rank;
    private String processName;
    private String processDisplayName;
    private String category;
    private Long totalDurationSecs;
    private String durationText;
    private Integer deviceCount;
    private Long avgDurationSecs;
    private Integer openCount;
}

// ClassSoftwareOverviewVO.java
@Data
public class ClassSoftwareOverviewVO {
    private Long classId;
    private String className;
    private LocalDate date;
    private Integer deviceCount;
    private Integer onlineDeviceCount;
    private Integer activeDeviceCount;
    private Double totalUsageHours;
    private List<SoftwareStatVO> topSoftware;
    private List<DeviceStatusVO> deviceStatusList;
}

@Data
public class DeviceStatusVO {
    private Long deviceId;
    private String deviceName;
    private Boolean online;
    private String currentProcess;
    private String currentWindowTitle;
    private Long todayUsageSecs;
}
```

## MyBatis XML 示例

### SoftwareUsageDetailMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xxx.hc.modules.software.dal.mapper.SoftwareUsageDetailMapper">

    <select id="selectPage" resultType="com.xxx.hc.modules.software.dal.vo.SoftwareUsageVO">
        SELECT
            sud.*,
            cd.name as deviceName,
            cc.className,
            cd.classroom_name as classroomName
        FROM software_usage_detail sud
        LEFT JOIN inspection_device cd ON sud.device_id = cd.id
        LEFT JOIN hc_school_class cc ON sud.class_id = cc.id
        <where>
            sud.deleted = 0
            <if test="req.deviceId != null">
                AND sud.device_id = #{req.deviceId}
            </if>
            <if test="req.classId != null">
                AND sud.class_id = #{req.classId}
            </if>
            <if test="req.processName != null and req.processName != ''">
                AND sud.process_name LIKE CONCAT('%', #{req.processName}, '%')
            </if>
            <if test="req.startTime != null">
                AND sud.start_time &gt;= #{req.startTime}
            </if>
            <if test="req.endTime != null">
                AND sud.start_time &lt;= #{req.endTime}
            </if>
        </where>
        ORDER BY sud.start_time DESC
    </select>

    <insert id="insertOrUpdate" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO software_usage_detail (
            session_id, device_id, device_code, process_name, window_title,
            exe_path, start_time, end_time, duration_secs, dept_id, class_id,
            tenant_id, creator, create_time
        ) VALUES (
            #{sessionId}, #{deviceId}, #{deviceCode}, #{processName}, #{windowTitle},
            #{exePath}, #{startTime}, #{endTime}, #{durationSecs}, #{deptId}, #{classId},
            #{tenantId}, #{creator}, NOW()
        ) ON DUPLICATE KEY UPDATE
            end_time = VALUES(end_time),
            duration_secs = VALUES(duration_secs),
            window_title = VALUES(window_title),
            updater = VALUES(creator),
            update_time = NOW()
    </insert>
</mapper>
```

## Controller 代码

### SoftwareUsageController.java（管理端）

```java
@RestController
@RequestMapping("/admin-api/hc/software-usage")
@Validated
public class SoftwareUsageController {

    @Resource
    private SoftwareUsageService softwareUsageService;

    @GetMapping("/page")
    @Operation(summary = "获取软件使用明细列表")
    public CommonResult<PageResult<SoftwareUsageVO>> page(SoftwareUsagePageReqDTO req) {
        return success(softwareUsageService.getUsagePage(req));
    }

    @GetMapping("/statistics/by-software")
    @Operation(summary = "按软件统计使用时长")
    public CommonResult<List<SoftwareStatVO>> statisticsBySoftware(SoftwareStatReqDTO req) {
        return success(softwareUsageService.getStatisticsBySoftware(req));
    }

    @GetMapping("/class-overview")
    @Operation(summary = "获取班级软件使用概览")
    public CommonResult<ClassSoftwareOverviewVO> classOverview(
            @RequestParam("classId") Long classId,
            @RequestParam(value = "date", required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate date) {
        return success(softwareUsageService.getClassOverview(classId, date));
    }
}
```

### ClientSoftwareUsageController.java（客户端推送）

```java
@RestController
@RequestMapping("/client/software/usage")
@Validated
@Slf4j
public class ClientSoftwareUsageController {

    @Resource
    private SoftwareUsagePushService pushService;

    @PostMapping("/realtime")
    @Operation(summary = "实时上报软件使用事件")
    public CommonResult<Boolean> realtime(@RequestBody @Valid SoftwareUsageRealtimeReqDTO req) {
        pushService.saveRealtimeUsage(req);
        return success(true);
    }

    @PostMapping("/batch")
    @Operation(summary = "批量上报软件使用记录")
    public CommonResult<Boolean> batch(@RequestBody @Valid SoftwareUsageBatchReqDTO req) {
        pushService.saveBatchUsage(req);
        return success(true);
    }
}
```

## Service 接口

```java
public interface SoftwareUsageService {
    PageResult<SoftwareUsageVO> getUsagePage(SoftwareUsagePageReqDTO req);
    List<SoftwareStatVO> getStatisticsBySoftware(SoftwareStatReqDTO req);
    ClassSoftwareOverviewVO getClassOverview(Long classId, LocalDate date);
}

public interface SoftwareUsagePushService {
    void saveRealtimeUsage(SoftwareUsageRealtimeReqDTO req);
    void saveBatchUsage(SoftwareUsageBatchReqDTO req);
}
```

## 数据聚合定时任务

```java
@Component
@Slf4j
public class SoftwareUsageAggregationJob {

    @Resource
    private SoftwareUsageDetailMapper detailMapper;
    @Resource
    private SoftwareUsageDailyMapper dailyMapper;

    @Scheduled(cron = "0 30 23 * * ?")
    public void aggregateDailyUsage() {
        // 1. 聚合今日数据到日汇总表
        // 2. 删除30天前的明细数据
        LocalDate yesterday = LocalDate.now().minusDays(1);

        // 聚合逻辑：按 device_id + process_name + usage_date 分组统计
        // INSERT INTO software_usage_daily ...
        // SELECT device_id, process_name, DATE(start_time) as usage_date,
        //        SUM(duration_secs) as total_duration_secs,
        //        COUNT(*) as open_count
        // FROM software_usage_detail
        // WHERE DATE(start_time) = yesterday
        // GROUP BY device_id, process_name, DATE(start_time)
        // ON DUPLICATE KEY UPDATE ...

        // 清理30天前的明细数据
        LocalDateTime cutoff = LocalDateTime.now().minusDays(30);
        detailMapper.deleteByStartTimeBefore(cutoff);
    }
}
```

## 注意事项

1. **数据一致性**：明细表使用 `session_id` 作为唯一键，支持幂等写入
2. **性能优化**：明细表保留30天，查询性能通过索引优化
3. **租户隔离**：所有查询必须带上 `tenant_id` 条件
4. **软删除**：使用 `deleted` 字段实现软删除，不物理删除数据
5. **时间转换**：客户端推送的是时间戳（毫秒），服务端转换为 LocalDateTime

---

设计日期：2026-03-09
版本：v1.0
