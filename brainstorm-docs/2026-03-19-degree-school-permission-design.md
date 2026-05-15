# 学位类型新增学校审核/录取权限功能设计

> **日期**: 2026-03-19
> **功能**: 学位类型增加两个字段（是否允许学校审核、是否允许学校录取），实现学校用户、旗县区用户、盟市账号的招生审核流程和权限控制

---

## 一、业务背景

### 1.1 现有架构

- **缓存初始化**: `PortalCacheInitializer` 在项目启动时从 Redis 拉取数据到本地内存
- **缓存内容**: 报名类型、学位类型、字段配置、优待人群配置
- **Feign 调用**: 门户端通过 Feign 调用管理端 API 获取缓存数据

### 1.2 现有用户角色

| 角色 | 判断条件 | 说明 |
|-----|---------|------|
| 盟市账号 | `RoleTypeUtil.getDeptType()` 返回 STR1 | 最高级别，可查看所有 |
| 旗县区用户 | `RoleTypeUtil.getDeptType()` 返回 STR2 | 区教育局，可查看本区 |
| 学校用户 | `RoleTypeUtil.getDeptType()` 返回 STR3 | 学校管理员，按权限操作 |

---

## 二、功能需求

### 2.1 学位类型新增字段

**表**: `cfm_degree`

| 字段名 | 类型 | 默认值 | 说明 |
|-------|------|-------|------|
| iz_allow_school_check | VARCHAR(1) | '0' | 是否允许学校审核 |
| iz_allow_school_enroll | VARCHAR(1) | '0' | 是否允许学校录取 |

**实体类**: `CfmDegreeEntity.java`

```java
/**
 * 是否允许学校审核（默认否）
 * 0-否 1-是
 */
@Excel(name = "是否允许学校审核", width = 15, replace = {"是_1", "否_0"})
@ApiModelProperty(value = "是否允许学校审核")
private String izAllowSchoolCheck;

/**
 * 是否允许学校录取（默认否）
 * 0-否 1-是
 */
@Excel(name = "是否允许学校录取", width = 15, replace = {"是_1", "否_0"})
@ApiModelProperty(value = "是否允许学校录取")
private String izAllowSchoolEnroll;
```

### 2.2 字段联动规则

当 `izAllowSchoolCheck = '1'` 时，**必须**配置以下二选一：
- `izOpenGraduationSchoolApproval = '1'` - 开启毕业学校审批
- `izOpenAdmissionSchoolApproval = '1'` - 开启录取学校审批

---

## 三、权限矩阵

### 3.1 学校用户权限分配

| 配置 | 毕业学校审批 | 录取学校审批 |
|-----|------------|-------------|
| **毕业学校管理员** | 审核 + 查看 | 只能查看，**不能录取** |
| **公办/民办学校管理员** | 只能查看，**不能录取** | 审核 + 录取 + 查看 |

### 3.2 按钮显示逻辑

| 按钮 | 显示条件 |
|-----|---------|
| 审核按钮 | `izAllowSchoolCheck = '1'` 且 学生还没被审核 |
| 录取按钮 | `izAllowSchoolEnroll = '1'` 且 学生已审核通过但还没被录取 |

### 3.3 学校用户查看范围

学校管理员可查看以下学生：
- **报名了本校的学生**（`publicSchool` 或 `privateSchool` = 本校）
- **毕业于本校的学生**（`graduationSchool` = 本校）

但审核/录取权限根据配置决定。

### 3.4 旗县区/盟市用户

- **旗县区用户（STR2）**: 可审核/录取当前旗县区下所有学生
- **盟市账号（STR1）**: 可审核/录取所有旗县区下所有学生
- **幼教/普教筛选**: 通过 `eduType` 判断，幼教只看幼儿园，普教看幼升小和小升初

---

## 四、接口设计

### 4.1 新建接口

**接口路径**: `GET /portalStudentInfo/manager/schoolList`

**职责**: 供学校用户（公办/民办/毕业学校管理员）查看自己权限范围内的学生列表，并返回操作权限标识。

**Controller 方法**: `PortalStudentInfoAdminController.java`

```java
@GetMapping(value = "/schoolList")
public Result<IPage<PortalStudentInfoVO>> schoolList(
    PortalStudentInfoSearchVO portalStudentInfoSearchVO,
    @RequestParam(name = "pageNo", defaultValue = "1") Integer pageNo,
    @RequestParam(name = "pageSize", defaultValue = "10") Integer pageSize,
    HttpServletRequest req)
```

### 4.2 返回结构

```json
{
  "success": true,
  "result": {
    "records": [{
      "id": "xxx",
      "studentName": "张三",
      "graduationSchool": "毕业学校A",
      "graduationSchoolName": "xxx幼儿园",
      "publicSchool": "公办学校B",
      "publicSchoolName": "xxx小学",
      "privateSchool": "民办学校C",
      "privateSchoolName": "xxx外国语学校",
      "canCheck": true,
      "canEnroll": false,
      "checkStatus": "0",
      "enrollStatus": "0"
    }],
    "total": 100,
    "size": 10,
    "current": 1
  }
}
```

### 4.3 权限标识字段

| 字段 | 类型 | 说明 |
|-----|------|------|
| canCheck | Boolean | 是否可审核 |
| canEnroll | Boolean | 是否可录取 |

---

## 五、核心实现

### 5.1 Service 层新增方法

**接口**: `IPortalStudentInfoService.java`

```java
/**
 * 学校用户-查询学生列表（包含权限标识）
 */
IPage<PortalStudentInfoVO> selectSchoolListByPage(
    Page<PortalStudentInfoVO> page,
    PortalStudentInfoSearchVO searchVO,
    LoginUser sysUser,
    CfmDegreeEntity degreeConfig);
```

### 5.2 权限判断逻辑

```java
private void setPermissionFlags(
    PortalStudentInfoVO vo,
    CfmDegreeEntity degreeConfig,
    String schoolId,
    String schoolType) {

    vo.setCanCheck(false);
    vo.setCanEnroll(false);

    String checkStatus = vo.getCheckStatus();
    String enrollStatus = vo.getEnrollStatus();

    // ========== 审核权限判断 ==========
    if ("1".equals(degreeConfig.getIzAllowSchoolCheck())) {
        if ("0".equals(checkStatus)) {
            if ("1".equals(degreeConfig.getIzOpenGraduationSchoolApproval())) {
                if (schoolId.equals(vo.getGraduationSchool())) {
                    vo.setCanCheck(true);
                }
            }
            if ("1".equals(degreeConfig.getIzOpenAdmissionSchoolApproval())) {
                if ("公办".equals(schoolType) && schoolId.equals(vo.getPublicSchool())) {
                    vo.setCanCheck(true);
                }
                if ("民办".equals(schoolType) && schoolId.equals(vo.getPrivateSchool())) {
                    vo.setCanCheck(true);
                }
            }
        }
    }

    // ========== 录取权限判断 ==========
    if ("1".equals(degreeConfig.getIzAllowSchoolEnroll())) {
        if ("1".equals(checkStatus) && "0".equals(enrollStatus)) {
            if ("公办".equals(schoolType) && schoolId.equals(vo.getPublicSchool())) {
                vo.setCanEnroll(true);
            }
            if ("民办".equals(schoolType) && schoolId.equals(vo.getPrivateSchool())) {
                vo.setCanEnroll(true);
            }
        }
    }
}
```

### 5.3 查询条件

```xml
<select id="selectSchoolListPage" resultType="...PortalStudentInfoVO">
    SELECT ... FROM portal_student_info
    WHERE del_flag = '0'
    AND year = #{year}
    AND (
        graduation_school = #{graduationSchoolId}
        <if test="publicSchoolId != null">
            OR public_school = #{publicSchoolId}
        </if>
        <if test="privateSchoolId != null">
            OR private_school = #{privateSchoolId}
        </if>
    )
</select>
```

---

## 六、文件清单

| 操作 | 文件路径 | 说明 |
|-----|---------|------|
| 修改 | `cfm_degree` 表 | 新增 `iz_allow_school_check` 和 `iz_allow_school_enroll` 字段 |
| 修改 | `CfmDegreeEntity.java` | 新增 `izAllowSchoolCheck`、`izAllowSchoolEnroll` 字段 |
| 修改 | `PortalStudentInfoVO.java` | 新增 `canCheck`、`canEnroll` 字段 |
| 修改 | `PortalStudentInfoSearchVO.java` | 新增 `graduationSchoolId`、`publicSchoolId`、`privateSchoolId` 字段 |
| 修改 | `IPortalStudentInfoService.java` | 新增 `selectSchoolListByPage` 方法 |
| 修改 | `PortalStudentInfoServiceImpl.java` | 实现 `selectSchoolListByPage` 方法 |
| 修改 | `PortalStudentInfoAdminController.java` | 新增 `/schoolList` 接口 |
| 修改 | `PortalStudentInfoMapper.xml` | 新增 `selectSchoolListPage` SQL |

---

## 七、实施步骤

### Step 1: 数据库改动

```sql
ALTER TABLE cfm_degree ADD COLUMN iz_allow_school_check VARCHAR(1) DEFAULT '0' COMMENT '是否允许学校审核';
ALTER TABLE cfm_degree ADD COLUMN iz_allow_school_enroll VARCHAR(1) DEFAULT '0' COMMENT '是否允许学校录取';
```

### Step 2: 实体类改动

- `CfmDegreeEntity.java` 新增字段
- `PortalStudentInfoVO.java` 新增权限字段
- `PortalStudentInfoSearchVO.java` 新增查询字段

### Step 3: Service 层

- 接口新增方法签名
- 实现类新增权限判断逻辑

### Step 4: Mapper 层

- XML 新增查询 SQL

### Step 5: Controller 层

- 新增 `/schoolList` 接口

### Step 6: 测试验证

- 学校用户登录 → 查看列表 → 验证权限显示正确

---

## 八、前端配合

前端根据后端返回的 `canCheck` 和 `canEnroll` 字段来显示/隐藏按钮：

```html
<!-- 审核按钮 -->
<a-button v-if="record.canCheck" type="primary" @click="handleCheck">
  审核
</a-button>

<!-- 录取按钮 -->
<a-button v-if="record.canEnroll" type="primary" @click="handleEnroll">
  录取
</a-button>
```

---

## 九、边界情况

### 9.1 毕业学校和报名学校是同一所

如果学生毕业学校和报名学校是同一所学校：
- 毕业学校审批模式下：学校管理员可以审核，但**不能录取**
- 录取学校审批模式下：学校管理员可以审核**也可以录取**

### 9.2 缓存刷新

当管理员修改了 `izAllowSchoolCheck` 或 `izAllowSchoolEnroll` 配置后，需要调用缓存刷新接口：

```
POST /enrollment/cache/refresh/{year}
```

### 9.3 现有接口不受影响

- `/list` 接口：继续给旗县区/盟市用户使用，已有幼教/普教筛选逻辑
- `/schoolList` 接口：新建，供学校用户使用

---

## 十、幼教/普教说明

### 10.1 学校用户

- `/schoolList` 接口仅限学校用户
- 混合学校（同时有幼儿园和小学）的情况不存在
- 学校管理员只管理一种类型的学校

### 10.2 旗县区/盟市用户

- 现有 `/list` 接口已有幼教/普教筛选逻辑
- 幼教用户（`eduType=1`）：只看幼儿园
- 普教用户（`eduType=2`）：看幼升小和小升初

---

## 十一、审核/录取操作

现有接口复用：
- `/checkStudentInfo` - 审核接口
- `/enrollStudentInfo` - 录取接口

操作日志记录使用现有的 `OperatorLog` 机制。
