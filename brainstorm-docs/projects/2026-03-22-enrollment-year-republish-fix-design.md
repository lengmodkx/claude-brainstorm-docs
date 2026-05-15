# 报名年度重新发布 Bug 修复设计

**日期**: 2026-03-22
**项目**: JeecgBoot202407 / jeecg-module-admin
**分支**: feature_update

---

## 问题描述

在 `/admin/cfmEnrollmentYear` 页面中，重新发布已停用的报名年度（如 2025 年）后：
1. 报名类型的 `iz_open` 状态没有正确切换（2026 年仍为开启，2025 年仍为关闭）
2. 学位类型没有变化
3. 门户端查询出来的仍然是旧年份（2026）的数据

---

## 根因分析

### 根因 1：`initializeForYear` 幂等检查跳过了 `switchToYear`

**文件**: `CfmSignUpTypeServiceImpl.java:43-51`

```java
public void initializeForYear(String year) {
    long count = this.lambdaQuery().eq(CfmSignUpType::getYear, year).count();
    if (count > 0) {
        log.warn("年度 {} 的报名类型已存在，跳过初始化", year);
        return;  // ← 直接返回，switchToYear(year) 被跳过！
    }
    // ...
    switchToYear(year);  // 只有新建数据时才调用
}
```

重新发布已有数据的年度时，`count > 0` 直接 return，`switchToYear` 不执行。

`switchToYear` 负责：
- `closeAllYears()`: 所有 `iz_open='0'` → `'1'`（关闭）
- `openYear(year)`: 指定年度 `iz_open='0'`（开启）

### 根因 2：年度切换后没有清理缓存

系统有两层缓存：
- **Admin Redis 缓存** (`EnrollmentCacheServiceImpl`) — key 格式 `enrollment:{year}:*`
- **Portal 本地内存缓存** (`PortalEnrollmentCache`) — JVM 级 `ConcurrentHashMap`

发布年度后两层缓存均未清理，门户端继续命中旧年份缓存。

---

## 修复方案

### 修复 1：在 `publish()` 同步流程中调用 `switchToYear`

**修改文件**: `CfmEnrollmentYearServiceImpl.java`

在停用旧年份之后，同步调用 `signUpTypeService.switchToYear()`：

```java
// 4. 停用其他已发布年份（现有逻辑）
for (CfmEnrollmentYear item : publishedList) { ... }

// 5. [新增] 同步切换报名类型开启状态
signUpTypeService.switchToYear(String.valueOf(newYear));
```

### 修复 2：清理旧年份和新年份的双层缓存

**修改文件**: `CfmEnrollmentYearServiceImpl.java`

新增注入：
- `EnrollmentCacheService`
- `RedisTemplate<String, String>`

在 `switchToYear` 之后：

```java
// 清理旧年份缓存
for (CfmEnrollmentYear item : publishedList) {
    Integer oldYear = item.getYear();
    enrollmentCacheService.cleanAll(oldYear);
    redisTemplate.convertAndSend("portal:cache:clear", "CLEAR_ALL:" + oldYear);
}

// 清理新年份缓存（让下次查询从 DB 重建）
enrollmentCacheService.cleanAll(newYear);
redisTemplate.convertAndSend("portal:cache:clear", "CLEAR_ALL:" + newYear);
```

Redis Pub/Sub 频道 `portal:cache:clear` 已由 `PortalCacheMessageListener` 监听，
`CLEAR_ALL:year` 消息会触发 `PortalEnrollmentCache.clearAll(year)`。

---

## 涉及文件

| 文件 | 修改类型 |
|------|----------|
| `CfmEnrollmentYearServiceImpl.java` | 新增 `switchToYear` 调用 + 缓存清理逻辑 |

## 不需要修改

- `CfmSignUpTypeServiceImpl.initializeForYear` — 幂等检查逻辑保留不变
- `EnrollmentYearPublishService` — 异步任务逻辑不变
- `PortalCacheMessageListener` — 已支持 `CLEAR_ALL:year`，无需改动
