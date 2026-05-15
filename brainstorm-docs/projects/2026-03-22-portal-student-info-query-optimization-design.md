# PortalStudentInfo 查询优化设计文档

**日期**: 2026-03-22
**模块**: jeecg-module-portal / portalStudentInfo
**问题**: 查询列表接口慢，数据量 8w → 预计 20w

---

## 背景

`portal_student_info` 表当前 8w 条记录，预计增长到 20w。
所有角色（盟市管理员/旗县区管理员/学校管理员）共用一个 `/portalStudentInfo/manager/list` 接口。
UI 基本相同，不需要拆分接口和页面。

---

## 根本原因分析

| 优先级 | 问题 | 影响 |
|--------|------|------|
| 🔴 最严重 | `psi.*` 把 5 个 blob 字段全部读出来 | 每次分页都触发大量磁盘 IO |
| 🔴 严重 | 重复索引（public_school/private_school/student_id_card 各有两个） | 写入变慢，优化器可能选错 |
| 🟡 中等 | `otherwise` 分支学校角色用 OR 条件（graduation/public/private），index merge 慢 | 学校角色查询慢 |
| 🟡 中等 | `selectListByExample`（导出接口）还是旧版 OR JOIN | 导出时全表扫描 |

---

## 方案 B 实施计划

### Step 1：删除重复索引（DDL，无代码改动）

```sql
-- 删除重复的单列索引（复合索引已经覆盖了）
DROP INDEX psi_idx_public_school ON portal_student_info;
DROP INDEX psi_idx_private_school ON portal_student_info;
DROP INDEX psi_idx_id_card ON portal_student_info;
DROP INDEX psi_idx_submit_status ON portal_student_info;
```

保留的索引：
- `idx_public_school_submit_del (public_school, submit_status, del_flag)`
- `idx_private_school_submit_del (private_school, submit_status, del_flag)`
- `idx_student_id_card (student_id_card(50))`

### Step 2：修复 `selectListByPage` 中 UNION ALL 分支的 `psi.*`

**文件**: `PortalStudentInfoMapper.xml`

**改动**: 把 `when` 分支（izQxq=3 且 belongSchoolId 不为空）里的三个子查询从 `psi.*` 改为明确列出字段，去掉所有 blob 字段。

blob 字段不能在 select 里出现：
- `domicile_info`
- `domicile_file`
- `equity_file_ids`
- `no_equity_file_ids`
- `special_file_ids`

改为 null 占位（这些字段在列表页不展示，详情页单独查询）：
```sql
null as domicile_info,
null as domicile_file,
null as equity_file_ids,
null as no_equity_file_ids,
null as special_file_ids,
```

同时 `otherwise` 分支也要做同样处理：把 `psi.*` 换成明确字段列表。

### Step 3：把 `otherwise` 分支的 OR 条件改为 UNION ALL

**文件**: `PortalStudentInfoMapper.xml`

当前 `otherwise` 分支（学校角色且没有 belongSchoolId 时）走 `commonConditions`，里面有：

```xml
AND (
    psi.graduation_school = #{studentInfo.graduationSchoolId}
    OR psi.public_school = #{studentInfo.publicSchoolId}
    OR psi.private_school = #{studentInfo.privateSchoolId}
)
```

**改为**：在 `otherwise` 分支里，如果 `izQxq=3`，也拆成 UNION ALL（参考现有 `when` 分支的结构），否则走简单查询。

### Step 4：修复 `selectListByExample`（导出接口）

**文件**: `PortalStudentInfoMapper.xml`

`selectListByExample` 还是旧版本，用的是 OR JOIN：
```sql
left join cfm_school_info csi on (csi.id=psi.public_school or csi.id=psi.private_school)
```

改为参考 `selectListByPage` 的新逻辑，去掉 OR JOIN，改用 `EXISTS` 子查询或 UNION ALL 方式。

由于导出不分页，数据量大，这里用 UNION ALL 代价也大，建议改为：
- 去掉 JOIN，仅在需要 `schoolType` 过滤时用 `EXISTS` 子查询
- 其余条件直接用 `commonConditions` 里的逻辑

---

## 涉及文件

| 文件 | 改动类型 |
|------|---------|
| `PortalStudentInfoMapper.xml` | 核心 SQL 改动（Step 2/3/4） |
| DDL 执行（数据库直接操作） | 删除重复索引（Step 1） |

---

## 实际实现说明（与计划的差异）

### Step 2 修正
原计划 null 掉所有 5 个 blob 字段。实际上 `domicile_file`、`equity_file_ids`、`no_equity_file_ids`、`special_file_ids` 四个字段存储的是文件 ID 字符串（短文本），`batchLoadFiles` 方法需要读取这些字段。因此：
- **只 null 掉 `domicile_info`**（存储大量户籍表单数据，列表不需要）
- 其余 4 个 blob 字段保留（存储短字符串 ID，不影响查询效率）

### Step 3 简化
原计划改 `otherwise` 为 UNION ALL。实际上：
- `commonConditions` 中的 per-mode 条件（graduationMode/admissionMode/viewMode）已经能正确过滤
- 只需删除冗余的"学校用户多条件OR查询"块即可
- per-mode 的 `NOT IN ... OR school = X` 模式在单个索引字段上，MySQL 可以有效处理

## 验收标准

1. 分页查询接口耗时从当前基线下降 ≥ 50%
2. 学校管理员角色查询结果正确（graduation/public/private 三种模式）
3. 导出接口功能正常，导出数据完整
4. 删除重复索引后写入功能正常（提交、审核、录取）
