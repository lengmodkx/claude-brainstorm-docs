# PortalStudentInfo list 接口重构设计

## 概述

重构 `PortalStudentInfoAdminController.list()` 接口，整合 `schoolList` 接口的逻辑，统一处理不同角色（学校管理员、旗县区/盟市）的查询需求。

## 需求分析

### 现状问题
- 存在两个接口：`/list` 和 `/schoolList`
- `schoolList` 逻辑不完整，权限判定有误
- `list` 接口缺少学校角色的完整权限控制

### 目标
- 删除 `schoolList` 接口
- 在 `list` 接口中完整实现学校角色查询逻辑
- 根据学位配置动态控制查看/审核/录取权限

## 角色权限设计

### 1. 学校管理员（DeptType=3）

#### 权限判定逻辑

```
1. 检查 izAllowSchoolCheck / izAllowSchoolEnroll
   └── 都为 "0" → 返回空列表（学校无权限查看）

2. 确定审批模式（二选一）：
   ├── izOpenGraduationSchoolApproval = "1" → 毕业学校审批模式
   │   └── 查询条件：graduation_school = schoolId
   │   └── canCheck/canEnroll = true
   │
   ├── izOpenAdmissionSchoolApproval = "1" → 录取学校审批模式
   │   └── 查询条件：public_school = schoolId OR private_school = schoolId
   │   └── canCheck/canEnroll = true
   │
   └── 两者都为 "0" → 默认查看模式
       └── 查询条件：graduation_school = schoolId OR public_school = schoolId
                     OR private_school = schoolId
       └── canCheck/canEnroll = false
```

#### 数据来源
- `PortalEnrollmentCache.getDegrees(year, enrollId)` - 获取学位配置
- `PortalEnrollmentCache.getSignupTypes(year)` - 获取报名类型列表

### 2. 旗县区/盟市（DeptType=2）

#### 查询范围

| eduType | 角色 | 可查看报名类型 |
|---------|------|---------------|
| "1" | 幼教 | 幼儿园报名 |
| "2" | 普教 | 幼升小入学信息采集、小升初入学信息采集 |

## 架构设计

### 数据流

```
┌─────────────────────────────────────────┐
│  /list 接口                              │
└─────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  1. 获取用户角色 (DeptType)               │
│  2. 根据角色执行不同逻辑                  │
└─────────────────────────────────────────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   ┌───────┐   ┌───────┐   ┌───────┐
   │Dept=1 │   │Dept=2 │   │Dept=3 │
   │市级   │   │旗县区 │   │学校   │
   └───────┘   └───────┘   └───────┘
                  │           │
                  ▼           ▼
           ┌─────────┐   ┌─────────────┐
           │按eduType│   │检查学位配置   │
           │过滤     │   │确定审批模式   │
           └─────────┘   └─────────────┘
                               │
                               ▼
                        ┌─────────────┐
                        │构建查询条件  │
                        │执行分页查询  │
                        └─────────────┘
```

### 核心类设计

#### SchoolPermissionResult

```java
@Data
public class SchoolPermissionResult {
    private boolean canView;                    // 是否有查看权限
    private boolean canCheck;                   // 是否有审核权限
    private boolean canEnroll;                  // 是否有录取权限
    private SchoolViewMode viewMode;            // 查看模式
    private List<String> allowedSignUpTypes;    // 允许查看的报名类型
}

enum SchoolViewMode {
    GRADUATION_SCHOOL,   // 毕业学校审批模式
    ADMISSION_SCHOOL,    // 录取学校审批模式
    VIEW_ONLY            // 仅查看模式
}
```

## SQL 查询设计

### 学校角色查询条件

```sql
-- 毕业学校审批模式
WHERE graduation_school = #{schoolId}
  AND sign_up_name IN (允许的报名类型列表)

-- 录取学校审批模式
WHERE (public_school = #{schoolId} OR private_school = #{schoolId})
  AND sign_up_name IN (允许的报名类型列表)

-- 默认查看模式
WHERE (graduation_school = #{schoolId}
       OR public_school = #{schoolId}
       OR private_school = #{schoolId})
```

## 接口变更

### 删除
- `GET /schoolList` - 学校用户查询接口
- `PortalStudentInfoService.selectSchoolListByPage()`
- `PortalStudentInfoMapper.selectSchoolListPage()`

### 修改
- `GET /list` - 增加学校角色完整权限控制逻辑

## 错误处理

| 场景 | 处理方式 |
|------|----------|
| 学校未关联 | Result.error("未找到学校信息") |
| 无权限配置 | 返回空列表 |
| 缓存失效 | 降级查询数据库 |
| 数据库异常 | Result.error("查询失败") |

## 测试用例

### 学校角色测试

1. **无权限配置** - 返回空列表
2. **毕业学校审批模式** - 只查 graduation_school，canCheck=true
3. **录取学校审批模式** - 只查 public/private_school，canCheck=true
4. **默认查看模式** - 查所有关联，canCheck=false

### 旗县区测试

1. **幼教角色** - signUpNames=["幼儿园报名"]
2. **普教角色** - signUpNames=["幼升小", "小升初"]

## 相关文件

- `PortalStudentInfoAdminController.java` - Controller 层
- `PortalStudentInfoService.java` / `PortalStudentInfoServiceImpl.java` - Service 层
- `PortalStudentInfoMapper.xml` - SQL 映射
- `PortalEnrollmentCache.java` - 缓存工具
- `CfmDegreeEntity.java` - 学位配置实体
