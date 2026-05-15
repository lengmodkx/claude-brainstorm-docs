# 部门管理新增"学校类型"字段实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 在 /system/depart 页面新增"学校类型"字段，支持多选（幼儿园/小学/初中），并与"是否学校"字段实现联动显示/隐藏和必填验证。

**Architecture:** 使用逗号分隔字符串存储多选值（1,2,3），前端通过 watch 监听实现实时联动，后端增加数据校验确保数据完整性。

**Tech Stack:** Java 8, Spring Boot 2.7, MyBatis-Plus, Vue 3, TypeScript, Ant Design Vue

---

## 前置检查

### Task 0: 确认当前数据库字典是否存在

**目的:** 检查 `school_type` 字典是否已存在，避免重复创建

**Command:**
```bash
cd jeecgboot-sy3.7 && mysql -u root -p -e "SELECT dict_code FROM sys_dict WHERE dict_code='school_type';"
```

**Expected:**
- 如果不存在记录，继续 Task 1
- 如果已存在，跳过 Task 1，检查字典项是否完整

---

## 第一阶段：数据库变更

### Task 1: 创建 Flyway 迁移脚本添加字典数据

**Files:**
- Create: `jeecgboot-sy3.7/jeecg-module-system/jeecg-system-biz/src/main/resources/flyway/sql/mysql/V1.0.16__add_school_type_dict.sql`

**Step 1: 创建迁移脚本**

```sql
-- 添加学校类型字典
INSERT INTO sys_dict (id, dict_name, dict_code, description, del_flag, create_time, create_by)
VALUES (
    (SELECT REPLACE(UUID(), '-', '')),
    '学校类型',
    'school_type',
    '学校类型多选（1=幼儿园，2=小学，3=初中）',
    0,
    NOW(),
    'system'
);

-- 添加字典项
SET @dict_id = (SELECT id FROM sys_dict WHERE dict_code = 'school_type' LIMIT 1);

INSERT INTO sys_dict_item (id, dict_id, item_text, item_value, description, sort_order, status, del_flag, create_time, create_by) VALUES
((SELECT REPLACE(UUID(), '-', '')), @dict_id, '幼儿园', '1', '幼儿园', 1, 1, 0, NOW(), 'system'),
((SELECT REPLACE(UUID(), '-', '')), @dict_id, '小学', '2', '小学', 2, 1, 0, NOW(), 'system'),
((SELECT REPLACE(UUID(), '-', '')), @dict_id, '初中', '3', '初中', 3, 1, 0, NOW(), 'system');

-- 添加 school_type 字段到 sys_depart 表
ALTER TABLE sys_depart ADD COLUMN IF NOT EXISTS school_type VARCHAR(20) COMMENT '学校类型（1=幼儿园,2=小学,3=初中，多选逗号分隔）';
```

**Step 2: 验证脚本语法**

检查 SQL 语法正确性，确保可以执行。

**Step 3: Commit**

```bash
cd jeecgboot-sy3.7
git add jeecg-module-system/jeecg-system-biz/src/main/resources/flyway/sql/mysql/V1.0.16__add_school_type_dict.sql
git commit -m "feat(db): add school_type dictionary and field to sys_depart table"
```

---

## 第二阶段：后端开发

### Task 2: 修改 SysDepart 实体类

**Files:**
- Modify: `jeecgboot-sy3.7/jeecg-module-system/jeecg-system-biz/src/main/java/org/jeecg/modules/system/entity/SysDepart.java`

**Step 1: 在实体类中添加 schoolType 字段**

在 `izSchool` 字段后添加：

```java
/**
 * 是否学校
 */
@Excel(name="机构类别",width=15,replace = {"是_1","否_0"})
private String izSchool;

/**
 * 学校类型（多选，逗号分隔：1=幼儿园,2=小学,3=初中）
 */
@Excel(name="学校类型",width=15,dicCode="school_type")
@Dict(dicCode = "school_type")
private String schoolType;
```

**Step 2: 更新 equals 和 hashCode 方法**

在 `equals` 方法中添加：
```java
Objects.equals(izSchool, depart.izSchool) &&
Objects.equals(schoolType, depart.schoolType) &&
```

在 `hashCode` 方法中添加 `schoolType`：
```java
Objects.hash(super.hashCode(), id, parentId, departName,
        departNameEn, departNameAbbr, departOrder, description, orgCategory,
        orgType, orgCode, mobile, fax, address, memo, status,
        delFlag, createBy, createTime, updateBy, updateTime, tenantId, izSchool, schoolType);
```

**Step 3: Commit**

```bash
cd jeecgboot-sy3.7
git add jeecg-module-system/jeecg-system-biz/src/main/java/org/jeecg/modules/system/entity/SysDepart.java
git commit -m "feat(entity): add schoolType field to SysDepart"
```

---

### Task 3: 添加后端数据校验

**Files:**
- Modify: `jeecgboot-sy3.7/jeecg-module-system/jeecg-system-biz/src/main/java/org/jeecg/modules/system/controller/SysDepartController.java`

**Step 1: 在 add 方法中添加校验**

找到 `add` 方法，在保存前添加：

```java
@PostMapping(value = "/add")
public Result<SysDepart> add(@RequestBody SysDepart sysDepart) {
    // 校验学校类型
    if ("1".equals(sysDepart.getIzSchool())) {
        if (StringUtils.isEmpty(sysDepart.getSchoolType())) {
            return Result.error("学校类型不能为空");
        }
        // 验证值合法性
        String[] types = sysDepart.getSchoolType().split(",");
        for (String type : types) {
            if (!Arrays.asList("1", "2", "3").contains(type.trim())) {
                return Result.error("学校类型值无效");
            }
        }
    } else {
        // 非学校时清空学校类型
        sysDepart.setSchoolType(null);
    }

    // 原有逻辑继续...
    Result<SysDepart> result = new Result<SysDepart>();
    // ...
}
```

**Step 2: 在 edit 方法中添加相同校验**

找到 `edit` 方法，在更新前添加相同的校验逻辑。

**Step 3: 添加必要的 import**

```java
import org.apache.commons.lang.StringUtils;
import java.util.Arrays;
```

**Step 4: Commit**

```bash
cd jeecgboot-sy3.7
git add jeecg-module-system/jeecg-system-biz/src/main/java/org/jeecg/modules/system/controller/SysDepartController.java
git commit -m "feat(controller): add schoolType validation in SysDepartController"
```

---

## 第三阶段：前端开发

### Task 4: 修改表单数据定义

**Files:**
- Modify: `front/jeecg-front/src/views/system/depart/depart.data.ts`

**Step 1: 添加学校类型字段到表单 schema**

在 `izSchool` 字段后添加：

```typescript
{
  field: 'izSchool',
  label: '是否学校',
  component: 'RadioButtonGroup',
  componentProps: {
    options: [
      { value: '1', label: '是' },
      { value: '0', label: '否' },
    ],
    defaultValue: '0',
  },
},
{
  field: 'schoolType',
  label: '学校类型',
  component: 'Select',
  componentProps: {
    mode: 'multiple',
    placeholder: '请选择学校类型',
    options: [
      { value: '1', label: '幼儿园' },
      { value: '2', label: '小学' },
      { value: '3', label: '初中' },
    ],
  },
  // 默认不显示，通过联动控制
  ifShow: false,
  rules: [{ required: true, message: '学校类型不能为空', type: 'array' }],
},
```

**Step 2: Commit**

```bash
cd front/jeecg-front
git add src/views/system/depart/depart.data.ts
git commit -m "feat(form): add schoolType field to form schema"
```

---

### Task 5: 修改 DepartFormTab.vue 添加联动逻辑

**Files:**
- Modify: `front/jeecg-front/src/views/system/depart/components/DepartFormTab.vue`

**Step 1: 添加 watch 监听 izSchool 变化**

在 `onMounted` 中添加监听逻辑：

```typescript
// 监听"是否学校"变化，控制"学校类型"显示/隐藏和清空
watch(
  () => props.data?.izSchool,
  async (newVal) => {
    const isSchool = newVal === '1';

    if (!isSchool) {
      // 切换为"否"时，清空学校类型
      await setFieldsValue({ schoolType: undefined });
    }

    // 动态更新 schema 控制显示和验证
    await updateSchema([
      {
        field: 'schoolType',
        ifShow: isSchool,
        rules: isSchool
          ? [{ required: true, message: '学校类型不能为空', type: 'array', min: 1 }]
          : [],
      },
    ]);
  },
  { immediate: true }
);
```

**Step 2: 确保在 data 变化时也触发联动**

确认在 `watch(() => props.data, ...)` 中已经包含完整的字段回填逻辑。如果有需要，在数据加载后手动触发一次校验更新。

**Step 3: Commit**

```bash
cd front/jeecg-front
git add src/views/system/depart/components/DepartFormTab.vue
git commit -m "feat(form): add schoolType dynamic show/hide logic in DepartFormTab"
```

---

### Task 6: 修改 DepartFormModal.vue 添加联动逻辑

**Files:**
- Modify: `front/jeecg-front/src/views/system/depart/components/DepartFormModal.vue`

**Step 1: 在 registerModal 中添加联动初始化**

在 `setFieldsValue` 调用后添加：

```typescript
// 根据是否学校初始化学校类型的显示
const isSchool = record.izSchool === '1';
await updateSchema([
  {
    field: 'schoolType',
    ifShow: isSchool,
    rules: isSchool
      ? [{ required: true, message: '学校类型不能为空', type: 'array', min: 1 }]
      : [],
  },
]);

// 监听 izSchool 变化
watch(
  () => model.value.izSchool,
  async (newVal) => {
    const isSchoolNew = newVal === '1';

    if (!isSchoolNew) {
      // 切换为"否"时，清空学校类型
      await setFieldsValue({ schoolType: undefined });
    }

    await updateSchema([
      {
        field: 'schoolType',
        ifShow: isSchoolNew,
        rules: isSchoolNew
          ? [{ required: true, message: '学校类型不能为空', type: 'array', min: 1 }]
          : [],
      },
    ]);
  }
);
```

**注意:** 需要确保 `watch` 被正确导入和清理。

**Step 2: Commit**

```bash
cd front/jeecg-front
git add src/views/system/depart/components/DepartFormModal.vue
git commit -m "feat(modal): add schoolType dynamic logic in DepartFormModal"
```

---

## 第四阶段：测试验证

### Task 7: 后端测试

**Files:**
- Test via: API 测试工具或前端页面

**Step 1: 重启后端服务**

```bash
cd jeecgboot-sy3.7/jeecg-module-system/jeecg-system-start
mvn spring-boot:run
```

**Step 2: 验证数据库字段**

```sql
DESCRIBE sys_depart;
-- 确认 school_type 字段存在
```

**Step 3: 测试 API**

1. 测试新增部门（是学校，有学校类型）- 预期成功
2. 测试新增部门（是学校，无学校类型）- 预期失败，返回错误信息
3. 测试新增部门（不是学校）- 预期成功，school_type 为 null
4. 测试编辑部门，切换是否学校状态 - 验证联动逻辑

---

### Task 8: 前端测试

**Files:**
- Test via: 浏览器访问 http://localhost:3100/system/depart

**Step 1: 启动前端服务**

```bash
cd front/jeecg-front
pnpm dev
```

**Step 2: 测试场景**

| 场景 | 操作 | 预期结果 |
|------|------|----------|
| 1 | 打开新增弹窗 | "学校类型"字段不显示 |
| 2 | 选择"是否学校=是" | "学校类型"字段显示，为必填 |
| 3 | 选择几个学校类型后提交 | 提交成功，数据保存正确 |
| 4 | 切换"是否学校=否" | "学校类型"字段隐藏，值被清空 |
| 5 | 编辑已有学校数据的部门 | "学校类型"显示并回填正确值 |
| 6 | 在详情页查看 | "学校类型"显示为中文（如"幼儿园、小学"）|

---

## 第五阶段：收尾

### Task 9: 最终代码审查和提交

**Step 1: 检查所有修改**

```bash
# 后端
cd jeecgboot-sy3.7
git diff --stat

# 前端
cd front/jeecg-front
git diff --stat
```

**Step 2: 确保代码符合项目规范**

- Java 代码符合阿里巴巴 Java 开发规范
- TypeScript 代码通过 ESLint 检查
- 没有敏感信息泄露

**Step 3: 最终提交（如果还有未提交的更改）**

```bash
# 后端
cd jeecgboot-sy3.7
git add -A
git commit -m "feat(depart): complete schoolType field implementation"

# 前端
cd front/jeecg-front
git add -A
git commit -m "feat(depart): complete schoolType field UI implementation"
```

---

## 实施检查清单

- [ ] 数据库迁移脚本已创建并执行成功
- [ ] SysDepart 实体类已添加 schoolType 字段
- [ ] Controller 已添加校验逻辑
- [ ] 前端表单 schema 已添加 schoolType 字段
- [ ] DepartFormTab.vue 联动逻辑正常工作
- [ ] DepartFormModal.vue 联动逻辑正常工作
- [ ] 新增部门功能正常
- [ ] 编辑部门功能正常
- [ ] 联动显示/隐藏功能正常
- [ ] 必填验证功能正常

---

## 可能的问题及解决方案

### 问题1：Flyway 迁移脚本执行失败

**原因：** 可能是重复执行或 SQL 语法错误

**解决：**
- 检查 `flyway_schema_history` 表确认版本号
- 手动回滚后重新执行

### 问题2：前端联动不生效

**原因：** watch 监听未正确设置或 updateSchema 调用时机不对

**解决：**
- 检查 watch 是否正确监听 `props.data.izSchool`
- 确保 `immediate: true` 已设置
- 检查 `updateSchema` 是否是异步函数并正确使用 await

### 问题3：多选值提交格式不对

**原因：** Ant Design Vue Select 的 mode="multiple" 返回的是数组，但后端期望字符串

**解决：**
- 在提交前将数组转换为逗号分隔字符串
- 或在表单组件中添加 value formatter

### 问题4：编辑时 schoolType 不回显

**原因：** 后端返回的是逗号分隔字符串，前端 Select 需要数组

**解决：**
- 在 `setFieldsValue` 前将字符串转换为数组：`schoolType: record.schoolType?.split(',')`
- 或在提交时将数组转换为字符串

---

## 相关参考

- 部门列表 API: `/sysdepart/sysDepart/queryTreeList`
- 部门新增 API: `/sysdepart/sysDepart/add`
- 部门编辑 API: `/sysdepart/sysDepart/edit`
- 字典获取 API: `/sys/api/queryAllDictItems`
