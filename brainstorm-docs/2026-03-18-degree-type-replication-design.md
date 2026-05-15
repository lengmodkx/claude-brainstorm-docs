# 学位类型年度数据自动复制设计文档

**日期**: 2026-03-18
**主题**: 学位类型数据复制与默认配置管理
**状态**: 已确认

---

## 1. 需求概述

### 1.1 业务场景
- 学年管理发布新报名学年时，需要自动初始化学位管理数据
- 每个旗县区的每个报名类型都需要创建对应的学位类型数据
- 新服务器部署时，数据库无历史数据，需要本地默认配置兜底

### 1.2 核心需求
1. **数据复制**：优先从数据库上一年度复制学位类型配置
2. **默认兜底**：新系统无历史数据时，从本地 JSON 加载默认配置
3. **失效管理**：发布新学年时，上一年度学位数据自动失效
4. **配置导出**：将当前系统数据导出为 JSON 配置文件，随代码部署

---

## 2. 数据模型分析

### 2.1 实体关系

```
CfmDegreeEntity (学位类型主表)
├── id: 主键
├── sysOrgCode: 旗县区编码
├── enrollType: 报名类型（幼儿园/幼升小/小升初）
├── year: 年份
├── applyStartTime/applyEndTime: 报名起止时间
├── checkStartTime/checkEndTime: 审核起止时间
├── adjustStartTime/adjustEndTime: 调剂起止时间
├── izOpenSecondSchool: 是否开启第二志愿
├── izOpenPrivateSchool: 是否开启民办学校
├── izOpenGraduationSchool: 是否开启毕业学校
└── CfmDegreeInfo[] (学位详情列表)
    ├── degreeId: 关联主表ID
    ├── degreeName: 学位名称（A类/B类/C类等11种）
    ├── degreeNickName: 学位别称
    ├── sortNum: 排序
    ├── validType: 验证类型
    └── year: 年份
```

### 2.2 二维配置特性
- **维度1**：旗县区（orgCode）- 不同旗县区配置不同
- **维度2**：报名类型（enrollType）- 幼儿园/幼升小/小升初各有配置
- **组合数**：N个旗县区 × 3种报名类型 = 3N套配置

---

## 3. 架构设计

### 3.1 数据流转图

```
学年发布触发 (CfmEnrollmentYearServiceImpl.publish)
    │
    ▼
┌─────────────────────────────────────┐
│  YearDataReplicationService         │
│  （扩展现有服务）                      │
│                                     │
│  1. 字段配置复制（已存在）              │
│  2. 优待人群复制（已存在）              │
│  3. 报名类型初始化（已存在）             │
│  4. 【新增】学位类型初始化              │
│     ├── 失效上一年度数据               │
│     ├── 获取所有旗县区                 │
│     ├── 获取当前年度报名类型            │
│     └── 为每个(旗县区,报名类型)组合     │
│         复制/创建学位类型              │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  DegreeTypeReplicationService       │
│  （新增核心服务）                      │
│                                     │
│  数据获取优先级：                      │
│  1. 数据库上一年度（优先）              │
│  2. 本地 JSON 配置（兜底）             │
│  3. 系统硬编码默认（最终兜底）          │
└─────────────────────────────────────┘
```

### 3.2 组件清单

| 组件 | 类型 | 说明 |
|------|------|------|
| DegreeTypeReplicationService | Service | 核心复制逻辑，含失效与创建 |
| DegreeDataExportService | Service | 一次性导出工具 |
| DegreeDefaultDataLoader | Component | JSON 配置加载器 |
| degree-type-default.json | Resource | 默认配置文件 |

---

## 4. 数据结构与存储

### 4.1 JSON 配置文件结构（全量复制）

```json
{
  "version": "2025",
  "description": "学位类型默认配置 - 全量导出",
  "exportDate": "2026-03-18",
  "data": [
    {
      "orgCode": "150102",
      "orgName": "新城区",
      "enrollType": "幼儿园报名",
      "year": "2025",
      "degreeEntity": {
        "applyStartTime": null,
        "applyEndTime": null,
        "checkStartTime": null,
        "checkEndTime": null,
        "adjustStartTime": null,
        "adjustEndTime": null,
        "izOpenSecondSchool": "0",
        "izOpenPrivateSchool": "1",
        "izOpenGraduationSchool": "1"
      },
      "degreeInfos": [
        {"degreeName": "A类：有户有房", "degreeNickName": "A类", "sortNum": "1", "validType": "1"},
        {"degreeName": "B类：无户有房", "degreeNickName": "B类", "sortNum": "2", "validType": "3"},
        {"degreeName": "C类：有户无房", "degreeNickName": "C类", "sortNum": "3", "validType": "2"},
        {"degreeName": "D类：无户租房", "degreeNickName": "D类", "sortNum": "4", "validType": "4"},
        {"degreeName": "E类：有户租房", "degreeNickName": "E类", "sortNum": "5", "validType": "2"},
        {"degreeName": "F类：无户务工租房", "degreeNickName": "F类", "sortNum": "6", "validType": "4"},
        {"degreeName": "G类：学区有户无房", "degreeNickName": "G类", "sortNum": "7", "validType": "2"},
        {"degreeName": "有户祖房", "degreeNickName": "有户祖房", "sortNum": "8", "validType": "1"},
        {"degreeName": "无户祖房", "degreeNickName": "无户祖房", "sortNum": "9", "validType": "3"},
        {"degreeName": "旗直学籍有房", "degreeNickName": "旗直学籍有房", "sortNum": "10", "validType": "3"},
        {"degreeName": "旗直学籍无房", "degreeNickName": "旗直学籍无房", "sortNum": "11", "validType": "4"}
      ]
    }
  ]
}
```

### 4.2 文件存储位置

```
jeecg-module-admin-biz/src/main/resources/default-data/
├── field-config-default.json      # 已存在 - 动态字段默认配置
├── priority-group-default.json    # 已存在 - 优待人群默认配置
└── degree-type-default.json       # 【新增】- 学位类型默认配置
```

---

## 5. 核心逻辑详解

### 5.1 主流程：replicateDegreeTypesForYear

```java
@Transactional(rollbackFor = Exception.class)
public void replicateDegreeTypesForYear(Integer newYear) {
    // 1. 幂等性检查 - 避免重复初始化
    if (hasDataForYear(newYearStr)) {
        log.info("年度 [{}] 已存在学位类型数据，跳过初始化", newYear);
        return;
    }

    // 2. 失效上一年度数据
    Integer oldYear = newYear - 1;
    invalidateOldYearData(oldYear);

    // 3. 获取旗县区和报名类型
    List<String> orgCodes = getAllOrgCodes();
    List<CfmSignUpType> activeSignUpTypes = getActiveSignUpTypes(newYearStr);

    // 4. 为每个(旗县区, 报名类型)组合创建学位类型
    for (String orgCode : orgCodes) {
        String orgName = getOrgNameByCode(orgCode);
        for (CfmSignUpType signUpType : activeSignUpTypes) {
            createDegreeTypeForOrgAndEnrollType(
                newYear, orgCode, orgName, signUpType.getSignUpName());
        }
    }
}
```

### 5.2 数据获取优先级

```
┌─────────────────────────────────────────┐
│           创建新年度学位类型              │
│   createDegreeTypeForOrgAndEnrollType   │
└─────────────────────────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│ 从数据库 │   │ 从JSON  │   │ 系统默认 │
│上一年度  │   │本地配置  │   │ 硬编码   │
│  复制   │   │  加载   │   │  兜底   │
└─────────┘   └─────────┘   └─────────┘
     │             │             │
     └─────────────┴─────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│       保存新数据（主表 + 详情表）         │
└─────────────────────────────────────────┘
```

### 5.3 失效逻辑：invalidateOldYearData

```java
public void invalidateOldYearData(Integer year) {
    String yearStr = year.toString();

    // 1. 失效学位类型主表（逻辑删除）
    LambdaUpdateWrapper<CfmDegreeEntity> degreeWrapper =
        new LambdaUpdateWrapper<>();
    degreeWrapper.eq(CfmDegreeEntity::getYear, yearStr)
                 .eq(CfmDegreeEntity::getDelFlag, "0")
                 .set(CfmDegreeEntity::getDelFlag, "1");
    int degreeCount = degreeService.update(degreeWrapper);

    // 2. 失效学位类型详情表（逻辑删除）
    LambdaUpdateWrapper<CfmDegreeInfo> infoWrapper =
        new LambdaUpdateWrapper<>();
    infoWrapper.eq(CfmDegreeInfo::getYear, yearStr)
               .eq(CfmDegreeInfo::getDelFlag, "0")
               .set(CfmDegreeInfo::getDelFlag, "1");
    int infoCount = degreeInfoService.update(infoWrapper);

    log.info("失效 [{}] 年度学位数据：主表{}条，详情{}条",
        year, degreeCount, infoCount);
}
```

---

## 6. 实现文件清单

| 文件路径 | 类型 | 说明 |
|----------|------|------|
| `org.jeecg.modules.admin.enrollment.service.DegreeTypeReplicationService` | Class | 核心复制服务 |
| `org.jeecg.modules.admin.enrollment.service.DegreeDataExportService` | Class | 导出工具服务 |
| `org.jeecg.modules.admin.enrollment.config.DegreeDefaultDataLoader` | Class | 配置加载组件 |
| `resources/default-data/degree-type-default.json` | Resource | 默认配置文件 |
| `test/.../DegreeDataExportTest` | Test | 导出测试类 |

---

## 7. 使用流程

### 7.1 首次导出（当前系统）

```java
// 运行导出测试
@Test
public void exportCurrentConfig() {
    String json = exportService.exportCurrentConfigToJson();
    exportService.exportAndSaveToFile(
        "src/main/resources/default-data/degree-type-default.json");
}
```

### 7.2 代码提交

```bash
git add src/main/resources/default-data/degree-type-default.json
git commit -m "feat(degree-type): 添加学位类型默认配置"
```

### 7.3 新服务器部署

1. 部署代码（包含 degree-type-default.json）
2. 首次发布学年时会自动从 JSON 加载配置
3. 后续年度发布从数据库上一年复制

### 7.4 年度发布调用链

```
CfmEnrollmentYearServiceImpl.publish(yearId)
    ├── YearDataReplicationService.replicateDataForYear(newYear)
    │   ├── 复制字段配置
    │   └── 复制优待人群
    ├── ICfmSignUpTypeService.initializeForYear(year)
    │   └── 初始化报名类型
    └── DegreeTypeReplicationService.replicateDegreeTypesForYear(newYear)  【新增】
        ├── invalidateOldYearData(oldYear)      【失效旧数据】
        └── createDegreeTypes(...)               【创建新数据】
            ├── 从数据库复制（优先）
            └── 或从 JSON 加载（兜底）
```

---

## 8. 关键设计决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 数据存储格式 | JSON | 与现有 default-data 机制保持一致 |
| JSON 结构 | 全量复制 | 精确保留当前系统所有配置 |
| 复制策略 | DB优先，JSON兜底 | 正常用历史数据，新服务器用默认配置 |
| 失效方式 | 逻辑删除（delFlag） | 保留历史数据可追溯 |
| 导出时机 | 一次性静态导出 | 简单可控，随代码版本管理 |
| 事务边界 | 单年度操作原子化 | 失败自动回滚，保证数据一致性 |

---

## 9. 验收标准

- [ ] 新服务器首次发布学年能正确从 JSON 加载学位类型配置
- [ ] 正常运行时能从数据库上一年复制学位类型配置
- [ ] 发布新学年能正确失效上一年度所有学位数据
- [ ] 每个(旗县区, 报名类型)组合都有正确的学位类型数据
- [ ] 事务一致性：失败时数据自动回滚，不出现脏数据

---

## 10. 后续扩展建议

1. **配置管理界面**：提供可视化界面查看和编辑默认配置
2. **配置版本管理**：支持多版本默认配置，按年度切换
3. **增量导出**：支持导出特定旗县区的配置变更
4. **配置验证**：启动时验证 JSON 配置完整性

---

**设计确认日期**: 2026-03-18
**下一步**: 进入实现阶段，创建具体代码文件
