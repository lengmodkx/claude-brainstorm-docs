# 报名年份管理模块设计方案

## 概述

为 JeecgBoot 系统增加报名年份管理模块，支持多年度配置但仅单年度生效，包含报名起止时间设置，由市教育局统一管理。

**适用场景**：学生入学报名（小升初、初升高）

**核心特性**：
- 可预配置未来多年度（2025/2026/2027），系统自动判定当前生效年度
- 报名时间精确到时分秒
- 市教育局统一管理，学校/学生只读

---

## 数据库表结构

**表名**：`cfm_enrollment_year`

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | VARCHAR(36) | 主键ID，@TableId(type = IdType.ASSIGN_ID) |
| year | INT | 报名年度，如 2025，唯一索引 |
| year_name | VARCHAR(100) | 年度名称，如"2025年秋季入学报名" |
| start_time | DATETIME | 报名开始时间，精确到秒 |
| end_time | DATETIME | 报名结束时间，精确到秒 |
| status | TINYINT | 状态：0=草稿 1=已发布 2=已停用 |
| description | VARCHAR(500) | 备注说明 |
| create_by | VARCHAR(50) | 创建人 |
| create_time | DATETIME | 创建时间 |
| update_by | VARCHAR(50) | 更新人 |
| update_time | DATETIME | 更新时间 |
| sys_org_code | VARCHAR(50) | 所属部门编码（市教育局） |
| sys_company_code | VARCHAR(50) | 所属公司编码 |
| del_flag | TINYINT | 删除标记：0=正常 1=已删除 |

**索引设计**：
- 唯一索引：`year`（确保年度不重复）
- 普通索引：`status` + `start_time` + `end_time`（查询生效年度用）

---

## 后端设计

### 实体类

```java
@Data
@TableName("cfm_enrollment_year")
@EqualsAndHashCode(callSuper = true)
public class CfmEnrollmentYear extends JeecgEntity {

    @ApiModelProperty("报名年度")
    @NotNull(message = "报名年度不能为空")
    private Integer year;

    @ApiModelProperty("年度名称")
    @NotBlank(message = "年度名称不能为空")
    private String yearName;

    @ApiModelProperty("报名开始时间")
    @JsonFormat(timezone = "GMT+8", pattern = "yyyy-MM-dd HH:mm:ss")
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @NotNull(message = "开始时间不能为空")
    private Date startTime;

    @ApiModelProperty("报名结束时间")
    @JsonFormat(timezone = "GMT+8", pattern = "yyyy-MM-dd HH:mm:ss")
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @NotNull(message = "结束时间不能为空")
    private Date endTime;

    @ApiModelProperty("状态：0=草稿 1=已发布 2=已停用")
    private Integer status;

    @ApiModelProperty("备注说明")
    private String description;
}
```

### 核心服务方法

- `getCurrentYear()` - 获取当前生效的报名年度（供报名模块调用）
- `checkEnrollmentOpen()` - 检查当前是否在报名时间内
- `validateYearConflicts()` - 保存时校验时间冲突

### API 接口

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | `/cfm/enrollmentYear/list` | 查询列表 | 所有登录用户 |
| GET | `/cfm/enrollmentYear/current` | 获取当前生效年度 | 所有登录用户 |
| POST | `/cfm/enrollmentYear/add` | 新增 | 市教育局 |
| PUT | `/cfm/enrollmentYear/edit` | 编辑 | 市教育局 |
| PUT | `/cfm/enrollmentYear/publish/{id}` | 发布年度 | 市教育局 |
| PUT | `/cfm/enrollmentYear/disable/{id}` | 停用年度 | 市教育局 |
| DELETE | `/cfm/enrollmentYear/delete` | 删除 | 市教育局 |

### 权限控制

使用 JeecgBoot 的 `@RequiresPermissions` 注解：
- 市教育局角色：`cfm:enrollmentYear:all`（完整权限）
- 学校/学生：只允许查询接口，无修改权限

---

## 前端设计

### 管理页面（市教育局使用）

功能模块：
- **列表页**：展示所有年度配置，支持按年度、状态筛选
- **新增/编辑弹窗**：表单包含年度、年度名称、起止时间（日期时间选择器）、状态、备注
- **操作按钮**：发布/停用、编辑、删除（仅草稿状态可删）

### 表单校验规则

- 结束时间必须晚于开始时间
- 同一年度不能重复创建
- 已发布的年度修改时给出警告提示

### 组件选型（Ant Design Vue）

- 时间选择：`RangePicker` + `showTime` 显示时分秒
- 状态展示：`Badge` 组件区分颜色（绿色=已发布、灰色=草稿、红色=已停用）
- 权限控制：`v-has` 指令控制按钮显示

---

## 核心业务逻辑

### 获取当前生效年度的流程

```
请求 → 查询 status=1(已发布) 且 year=今年 的记录
    ↓
  检查当前时间是否在 [start_time, end_time] 范围内
    ↓
  是 → 返回该年度配置（报名开放）
  否 → 返回 null（报名未开放或已结束）
```

### 关键业务校验

**保存/修改时**：
- 同一年度只能有一条记录（唯一索引保证）
- 已发布的年度修改时，如果缩短时间导致当前时间超出范围，给出确认提示

**发布年度时**：
- 检查该年度时间范围是否合理（开始时间 < 结束时间）
- 检查是否与其他已发布年度的时间重叠
- 同一年度内只能有一个已发布的配置

### 缓存策略

使用 Redis 缓存当前生效年度：
- Key: `cfm:enrollmentYear:current`
- TTL: 5分钟
- 更新年度配置时清除缓存

---

## 错误处理

| 场景 | 错误码 | 提示信息 |
|------|--------|----------|
| 年度重复 | 400 | "该报名年度已存在" |
| 时间倒置 | 400 | "结束时间必须晚于开始时间" |
| 当前无生效年度 | 200 | 返回 null |
| 修改已发布年度 | 409 | "该年度已发布，修改会影响正在进行中的报名，是否继续？" |
| 删除已发布年度 | 403 | "已发布的年度不能删除" |

---

## 与报名模块集成

其他模块通过 `/cfm/enrollmentYear/current` 接口获取当前生效年度：

```java
// 报名前检查示例
CfmEnrollmentYear currentYear = enrollmentYearService.getCurrentYear();
if (currentYear == null || !currentYear.isInEnrollmentPeriod()) {
    throw new JeecgBootException("当前不在报名时间内");
}
```

---

## 验收标准

- [ ] 市教育局可以增删改查报名年度配置
- [ ] 可以预配置未来多年度（2025/2026/2027）
- [ ] 系统能正确识别当前生效的报名年度
- [ ] 报名模块能通过接口获取当前生效年度进行校验
- [ ] 时间精确到秒，时区统一为 GMT+8
- [ ] 并发操作安全，数据一致性有保障

---

## 技术规范

- **表前缀**：`cfm_`
- **时区**：GMT+8
- **时间精度**：秒（yyyy-MM-dd HH:mm:ss）
- **状态定义**：0=草稿 1=已发布 2=已停用
- **删除方式**：逻辑删除（del_flag）
