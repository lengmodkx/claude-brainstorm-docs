# 教育统计大数据平台 - 统计页面重构设计方案

## 设计目标

重构 `/dashboard/analysis` 统计页面，打造数据大屏风格展示界面，支持盟市、旗县区、学校三种角色根据登录用户权限动态切换数据视图。

---

## 设计原则

1. **科技深蓝风格** - 深蓝背景 (#0a1628 → #0d1f3c) + 青色高亮 (#00d4ff)
2. **网格卡片式布局** - 每个图表独立卡片，多列网格排列
3. **图表+数据并排** - 左侧图表，右侧关键数据
4. **悬停查看明细** - 鼠标悬停显示详细数值

---

## 技术选型

| 技术 | 选择 |
|------|------|
| 图表库 | ECharts 5.5.1 |
| UI框架 | Ant Design Vue 4.2.5 |
| 布局 | 网格卡片 + Flex 响应式 |
| 配色 | CSS 变量统一管理 |

---

## 角色权限体系

### 角色类型定义

| 角色值 | 角色 | 数据范围 |
|--------|------|----------|
| 1 | 盟市教育局 | 全市所有旗县区、所有学校 |
| 2 | 旗县区教育局 | 本区内所有学校（按 eduType 过滤）|
| 3 | 学校 | 本校数据 |

### 教育类型 (eduType)

| 值 | 类型 | 旗县区可见范围 |
|----|------|----------------|
| 1 | 幼教 | 仅幼儿园 |
| 2 | 普教 | 幼儿园 + 小学 + 初中 |
| 3 | 管理员 | 全部学段 |

---

## 页面布局

### 顶部导航栏

```
┌──────────────────────────────────────────────────────────────┐
│  📊 教育统计大数据平台    [旗县区 ▼] / [学校标签]   2026-03-26 │
└──────────────────────────────────────────────────────────────┘
```

**角色差异：**
- 盟市：显示「旗县区切换下拉框」
- 旗县区：无下拉框，直接显示本区
- 学校：显示「学校名称标签」

### 主体内容

```
┌──────────────────────────────────────────────────────────────┐
│                    ◆ 角色标识 · 数据范围 ◆                     │
├──────────────────────────────────────────────────────────────┤
│  报名统计                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │各旗县区/ │ │ 各学段   │ │ 各学校   │ │ 审核情况 │       │
│  │各学校人数 │ │ 人数    │ │ (表格)   │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  学生画像                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 民族分布 │ │ 男女分布 │ │ 学位分布 │ │ 学位通过 │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  学校排行                                                     │
│  ┌─────────────────────┐ ┌─────────────────────────────┐     │
│  │    公办 TOP10       │ │        民办 TOP10           │     │
│  └─────────────────────┘ └─────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

---

## 角色数据视图

### 盟市教育局 (RoleType = 1)

**顶部：** 旗县区下拉框（默认「呼伦贝尔市」）

**报名统计：**
- 各旗县区人数（柱状图）
- 各学段人数（饼图）
- 各学段旗县区分布（堆叠柱状图）
- 全市录取审核情况

**学生画像：** 全市民族/男女/学位/通过率分布

**学校排行：** 公办/民办 TOP10

**切换旗县区后：** 数据变为该旗县区汇总数据

---

### 旗县区教育局 (RoleType = 2)

**顶部：** 无下拉框，显示「XX区教育局 · 本区统计」

**筛选标签：** 教育类型筛选（全部/幼儿园/小学/初中），管理员可见所有

**报名统计：**
- 各学校报名人数（表格）
- 各学段人数（柱状图）
- 本区学校审核情况
- 学校审核明细

**学生画像：** 本区民族/男女/学位/通过率分布

**学校排行：** 本区公办/民办 TOP

---

### 学校管理员 (RoleType = 3)

**顶部：** 学校名称标签（公办 · 初中）

**审核概览：**
- 四个状态卡片：总报名/待审核/已通过/未通过

**审核进度：**
- 进度条展示通过/待审/未通过比例
- 通过率大数字展示

**学生画像：** 本校民族/男女/年级/审核状态

**报名明细：** 班级报名表格 + 近期审核动态

---

## 统一视觉规范

### 配色系统

```css
:root {
    --theme-color: #00d4ff;        /* 主色调-青色 */
    --theme-dark: #0078d4;          /* 深青色 */
    --theme-glow: rgba(0, 212, 255, 0.3);  /* 发光效果 */
    --card-bg: linear-gradient(145deg, rgba(30, 58, 95, 0.6), rgba(13, 27, 42, 0.8));
    --card-border: rgba(0, 212, 255, 0.15);
}
```

### 背景

```css
background: linear-gradient(135deg, #0a1628 0%, #0d1f3c 50%, #0a1628 100%);
```

### 卡片样式

- 背景：`var(--card-bg)`
- 边框：`1px solid var(--card-border)`
- 圆角：`12px`
- 内边距：`20px`
- 顶部：`3px 渐变边框`

### 悬停效果

```css
:hover {
    border-color: rgba(0, 212, 255, 0.4);
    transform: translateY(-2px);
    box-shadow: 0 8px 32px rgba(0, 212, 255, 0.15);
}
```

---

## 13项统计指标归属

| # | 指标 | 盟市 | 旗县区 | 学校 |
|----|------|------|--------|------|
| 1 | 各旗县区有多少人 | ✓ | - | - |
| 2 | 各旗县区各学段人数 | ✓ | - | - |
| 3 | 各学校人数 | 切换后显示 | ✓ | - |
| 4 | 各学校各学段人数 | 切换后显示 | ✓ | - |
| 5 | 学校审核情况 | - | - | ✓ |
| 6 | 旗县区审核情况 | ✓ | ✓ | - |
| 7 | 全市录取审核情况 | ✓ | - | - |
| 8 | 学生民族分布 | ✓ | ✓ | ✓ |
| 9 | 学生男女分布 | ✓ | ✓ | ✓ |
| 10 | 学生学位分布 | ✓ | ✓ | ✓ |
| 11 | 学生学位通过分布 | ✓ | ✓ | ✓ |
| 12 | 公办报名TOP10 | ✓ | ✓ | - |
| 13 | 民办报名TOP10 | ✓ | ✓ | - |

---

## 设计文件

| 文件 | 说明 |
|------|------|
| `dashboard-design.html` | 盟市角色设计稿 |
| `dashboard-design-county.html` | 旗县区角色设计稿 |
| `dashboard-design-school.html` | 学校角色设计稿 |

---

## 接口设计

### 用户上下文获取

通过 Shiro SecurityUtils 获取当前登录用户：

```java
LoginUser sysUser = (LoginUser) SecurityUtils.getSubject().getPrincipal();
String orgCode = sysUser.getOrgCode();      // 组织编码
String eduType = sysUser.getEduType();      // 教育类型 (1幼教/2普教/3所有)
String deptType = RoleTypeUtil.getDeptType(orgCode, sysBaseApi);
// deptType: "1"=盟市, "2"=旗县区, "3"=学校
```

### 盟市角色 - 旗县区切换

盟市角色通过请求参数切换查看不同旗县区数据：

```
GET /api/stat/enrollment/byCounty?countyCode=150502
```

- 不传 `countyCode` → 返回全市汇总数据
- 传 `countyCode` → 返回该旗县区数据

### 接口列表

| 接口 | 方法 | 功能 | 权限 |
|------|------|------|------|
| `/api/stat/enrollment/overview` | GET | 报名统计概览 | 全部 |
| `/api/stat/enrollment/byCounty` | GET | 各旗县区人数统计 | 盟市 |
| `/api/stat/enrollment/byDegree` | GET | 各学段人数统计 | 全部 |
| `/api/stat/enrollment/bySchool` | GET | 各学校人数统计 | 盟市+旗县区 |
| `/api/stat/enrollment/schoolDetail` | GET | 学校详细（表格） | 旗县区 |
| `/api/stat/verify/overview` | GET | 审核统计概览 | 全部 |
| `/api/stat/verify/bySchool` | GET | 各学校审核情况 | 盟市+旗县区 |
| `/api/stat/verify/schoolDetail` | GET | 学校审核明细 | 旗县区+学校 |
| `/api/stat/profile/ethnicity` | GET | 民族分布 | 全部 |
| `/api/stat/profile/gender` | GET | 男女分布 | 全部 |
| `/api/stat/profile/degree` | GET | 学位分布 | 全部 |
| `/api/stat/profile/degreeVerify` | GET | 学位审核通过分布 | 全部 |
| `/api/stat/rank/public` | GET | 公办TOP10 | 盟市+旗县区 |
| `/api/stat/rank/private` | GET | 民办TOP10 | 盟市+旗县区 |

### 核心响应格式

#### 报名统计概览 `/api/stat/enrollment/overview`
```json
{
  "code": 200,
  "data": {
    "roleType": "1",
    "eduType": "2",
    "orgCode": "150700",
    "orgName": "呼伦贝尔市教育局",
    "totalEnroll": 73350,
    "totalSubmit": 32580,
    "pendingCount": 12450,
    "passedCount": 58200,
    "rejectedCount": 2700,
    "passedRate": 79.3,
    "verifyCompleteRate": 83.0
  }
}
```

#### 各旗县区人数统计 `/api/stat/enrollment/byCounty`
```json
{
  "code": 200,
  "data": {
    "selectedCounty": null,
    "selectedCountyName": "全市",
    "isAll": true,
    "total": 73350,
    "counties": [
      { "code": "150502", "name": "海拉尔区", "count": 8420, "publicCount": 6200, "privateCount": 2220 }
    ]
  }
}
```

#### 各学段人数统计 `/api/stat/enrollment/byDegree`
```json
{
  "code": 200,
  "data": {
    "countyCode": "150502",
    "countyName": "海拉尔区",
    "eduTypeFilter": "普教",
    "total": 4390,
    "degrees": [
      { "code": "1", "name": "幼儿园", "count": 1300 },
      { "code": "2", "name": "小学", "count": 1810 },
      { "code": "3", "name": "初中", "count": 1280 }
    ]
  }
}
```

#### 各学校人数统计 `/api/stat/enrollment/bySchool`
```json
{
  "code": 200,
  "data": {
    "countyName": "海拉尔区",
    "eduTypeFilter": "普教",
    "total": 4390,
    "schools": [
      { "code": "150502001", "name": "海拉尔区第一中学", "type": "public", "degreeCode": "3", "degreeName": "初中", "count": 1280 }
    ]
  }
}
```

#### 学校详细报名 `/api/stat/enrollment/schoolDetail`
```json
{
  "code": 200,
  "data": {
    "countyName": "海拉尔区",
    "eduTypeFilter": "全部",
    "total": 4390,
    "schools": [
      { "code": "150502001", "name": "海拉尔区第一中学", "type": "public", "degrees": [{ "name": "初中", "count": 1280 }], "total": 1280 }
    ]
  }
}
```

#### 审核统计概览 `/api/stat/verify/overview`
```json
{
  "code": 200,
  "data": {
    "roleType": "1",
    "orgName": "呼伦贝尔市教育局",
    "total": 73350,
    "pending": 12450,
    "passed": 58200,
    "rejected": 2700,
    "passedRate": 79.3,
    "verifyCompleteRate": 83.0
  }
}
```

#### 各学校审核情况 `/api/stat/verify/bySchool`
```json
{
  "code": 200,
  "data": {
    "countyName": "海拉尔区",
    "total": 4390,
    "schools": [
      { "code": "150502001", "name": "海拉尔区第一中学", "total": 1280, "pending": 120, "passed": 1100, "rejected": 60, "passedRate": 86.0 }
    ]
  }
}
```

#### 民族分布 `/api/stat/profile/ethnicity`
```json
{
  "code": 200,
  "data": {
    "total": 32580,
    "distribution": [
      { "name": "汉族", "count": 18000, "rate": 55.2 },
      { "name": "蒙古族", "count": 8500, "rate": 26.1 },
      { "name": "鄂温克族", "count": 2080, "rate": 6.4 },
      { "name": "其他", "count": 4000, "rate": 12.3 }
    ]
  }
}
```

#### 男女分布 `/api/stat/profile/gender`
```json
{
  "code": 200,
  "data": {
    "total": 32580,
    "male": 21080,
    "female": 11500,
    "ratio": "1.83:1"
  }
}
```

#### 学校排行榜 `/api/stat/rank/public`
```json
{
  "code": 200,
  "data": {
    "type": "public",
    "scope": "全市",
    "rankings": [
      { "rank": 1, "code": "150700001", "name": "呼伦贝尔市第一中学", "count": 2340 },
      { "rank": 2, "code": "150502001", "name": "海拉尔区第二中学", "count": 1990 }
    ]
  }
}
```

### 数据权限控制核心逻辑

```java
@GetMapping("/enrollment/bySchool")
public Result<List<SchoolStatVO>> getEnrollmentBySchool(
    @RequestParam(required = false) String countyCode
) {
    LoginUser sysUser = (LoginUser) SecurityUtils.getSubject().getPrincipal();
    String orgCode = sysUser.getOrgCode();
    String eduType = sysUser.getEduType();
    String deptType = RoleTypeUtil.getDeptType(orgCode, sysBaseApi);

    switch (deptType) {
        case "1": // 盟市
            if (StringUtils.isNotBlank(countyCode)) {
                return Result.ok(statService.getSchoolsByCounty(countyCode, eduType));
            } else {
                return Result.ok(statService.getAllCountiesStat());
            }
        case "2": // 旗县区
            return Result.ok(statService.getSchoolsByCounty(orgCode, eduType));
        case "3": // 学校
            return Result.ok(statService.getSchoolByCode(orgCode));
        default:
            return Result.error("无效的角色类型");
    }
}
```

---

## 后续计划

1. ✅ 页面设计稿完成（3个角色视图）
2. ✅ 接口设计完成
3. ⏳ 创建统计 Controller 和 Service
4. ⏳ 编写 Mapper XML 统计查询
5. ⏳ 前端组件开发
6. ⏳ 联调测试

---

*设计日期：2026-03-26*
*更新日期：2026-03-26（接口设计完成）*
