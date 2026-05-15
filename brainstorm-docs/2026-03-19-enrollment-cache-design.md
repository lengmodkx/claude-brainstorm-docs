# 招生系统缓存功能设计

## 需求背景

在项目初始化时，将当前系统中已发布年份所关联的动态字段配置信息、报名类型以及学位类型都更新到缓存中，并且在这些信息有改动时能够重新更新缓存。

门户端报名流程：用户选择学段 → 选择旗县区 → 选择报名类型，系统根据时间判断是否能报名。

## 设计方案

### 一、整体架构

#### 1.1 核心组件

| 组件 | 说明 |
|------|------|
| `EnrollmentCacheApplicationRunner` | 项目启动时初始化加载缓存 |
| `EnrollmentCacheService` | 缓存服务，提供查询和刷新方法 |
| `EnrollmentCacheCleaner` | 缓存清理工具类 |
| `EnrollmentCacheEventListener` | 事件监听器，监听数据变更事件触发缓存清理 |
| `PortalEnrollmentCache` | 门户端本地缓存管理 |
| `PortalCacheInitializer` | 门户端启动时从 Redis 拉取数据到本地 |

#### 1.2 缓存 Key 设计

```
enrollment:cache:{year}:signup_types                    # 某年份的报名类型列表
enrollment:cache:{year}:degrees:{enrollId}              # 某报名类型下的学位类型
enrollment:cache:{year}:degrees:info:{degreeId}         # 某学位详情
enrollment:cache:{year}:field_config:{orgCode}          # 某旗县区的字段配置
enrollment:cache:{year}:field_settings:{orgCode}       # 某旗县区的优待人群配置
```

#### 1.3 软刷新模式

- **读取时**：先查缓存，缓存不存在则查数据库并写入缓存
- **更新时**：直接删除缓存，下次查询时自动加载

### 二、事件机制设计

#### 2.1 数据变更事件类

| 事件类 | 说明 |
|--------|------|
| `SignupTypeChangedEvent` | 报名类型变更事件 |
| `DegreeChangedEvent` | 学位类型变更事件 |
| `DegreeInfoChangedEvent` | 学位详情变更事件 |
| `FieldConfigChangedEvent` | 字段配置变更事件 |
| `FieldSettingChangedEvent` | 优待人群变更事件 |

#### 2.2 事件发布点

在以下 Service 的增删改方法中发布事件：
- `CfmSignUpTypeServiceImpl` - 报名类型
- `CfmDegreeServiceImpl` - 学位类型
- `CfmDegreeInfoServiceImpl` - 学位详情
- `CfmFieldConfigServiceImpl` - 字段配置
- `CfmFieldSettingServiceImpl` - 优待人群

#### 2.3 关联缓存清理逻辑

| 事件 | 清理逻辑 |
|------|----------|
| `SignupTypeChangedEvent` | 清理该年份的报名类型缓存 + 关联的学位类型和详情 |
| `DegreeChangedEvent` | 清理该报名类型下的学位缓存 + 学位详情 |
| `DegreeInfoChangedEvent` | 清理该学位详情 |
| `FieldConfigChangedEvent` | 清理该旗县区的字段配置 |
| `FieldSettingChangedEvent` | 清理该旗县区的优待人群配置 |

### 三、Feign API 接口设计

#### 3.1 接口定义

```java
@FeignClient(name = "jeecg-admin", fallback = EnrollmentCacheApiFallback.class)
public interface EnrollmentCacheApi {

    @GetMapping("/enrollment/cache/signupTypes/{year}")
    Result<List<CfmSignUpType>> getSignupTypes(@PathVariable Integer year);

    @GetMapping("/enrollment/cache/degrees/{year}/{enrollId}")
    Result<List<CfmDegreeEntity>> getDegrees(@PathVariable Integer year,
                                             @PathVariable String enrollId);

    @GetMapping("/enrollment/cache/degreeInfo/{year}/{degreeId}")
    Result<CfmDegreeInfo> getDegreeInfo(@PathVariable Integer year,
                                        @PathVariable String degreeId);

    @GetMapping("/enrollment/cache/fieldConfig/{year}/{orgCode}")
    Result<CfmFieldConfig> getFieldConfig(@PathVariable Integer year,
                                          @PathVariable String orgCode);

    @GetMapping("/enrollment/cache/fieldSettings/{year}/{orgCode}")
    Result<List<CfmFieldSetting>> getFieldSettings(@PathVariable Integer year,
                                                   @PathVariable String orgCode);

    @PostMapping("/enrollment/cache/refresh/{year}")
    Result<Void> refreshAll(@PathVariable Integer year);
}
```

### 四、门户端本地缓存

#### 4.1 方案思路

1. **门户端启动时**从 Redis 一次性拉取所有已发布年份的配置数据到本地内存
2. **门户端本地**使用 `ConcurrentHashMap` 做本地缓存
3. **后续门户端查询**直接读本地，不走 Feign 调用
4. **数据变更时**通过消息通知门户端刷新本地缓存

#### 4.2 优势

- 查询毫秒级响应，无网络开销
- 系统高可用，不依赖管理端服务
- 数据变更时可快速刷新

### 五、代码目录结构

#### 5.1 管理端新增文件

```
jeecg-server-cloud/jeecg-module-admin/jeecg-module-admin-biz/src/main/java/org/jeecg/modules/admin/enrollment/
├── cache/
│   ├── EnrollmentCacheApplicationRunner.java
│   ├── EnrollmentCacheService.java
│   ├── EnrollmentCacheServiceImpl.java
│   ├── EnrollmentCacheCleaner.java
│   ├── controller/EnrollmentCacheController.java
│   └── event/
│       ├── SignupTypeChangedEvent.java
│       ├── DegreeChangedEvent.java
│       ├── DegreeInfoChangedEvent.java
│       ├── FieldConfigChangedEvent.java
│       ├── FieldSettingChangedEvent.java
│       └── EnrollmentCacheEventListener.java
```

```
jeecg-server-cloud/jeecg-module-admin/jeecg-module-admin-api/src/main/java/org/jeecg/modules/admin/enrollment/api/
├── EnrollmentCacheApi.java
└── fallback/EnrollmentCacheApiFallback.java
```

#### 5.2 门户端新增文件

```
jeecg-server-cloud/jeecg-module-portal/jeecg-module-portal-biz/src/main/java/org/jeecg/modules/portalStudentInfo/cache/
├── PortalEnrollmentCache.java
└── PortalCacheInitializer.java
```

#### 5.3 需要修改的现有文件

- `CfmSignUpTypeServiceImpl.java`
- `CfmDegreeServiceImpl.java`
- `CfmDegreeInfoServiceImpl.java`
- `CfmFieldConfigServiceImpl.java`
- `CfmFieldSettingServiceImpl.java`

### 六、实现顺序

1. 缓存服务基础实现（EnrollmentCacheService）
2. 事件类定义
3. 事件监听器
4. 修改现有 Service 添加事件发布
5. Feign API 接口
6. Controller 实现
7. 门户端本地缓存
8. 启动初始化加载

### 七、验收标准

1. 系统启动时自动加载已发布年份的配置数据到 Redis
2. 数据变更后对应缓存自动失效
3. 门户端可通过本地缓存高效查询数据
4. 其他微服务可通过 Feign API 获取缓存数据
