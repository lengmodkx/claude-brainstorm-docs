# 在线题库与考试系统设计

## 一、模块与题库管理

**模块命名：** `yudao-module-exam`

**题库实体设计：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| name | String | 题库名称 |
| description | String | 题库描述 |
| remark | String | 备注 |
| status | Integer | 状态（0=禁用 1=启用）。注：当前简化设计，如需草稿状态可扩展为 0=草稿 1=启用 2=禁用 |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |
| deleted | Boolean | 是否删除 |

**题目实体设计：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| bankId | Long | 题库ID |
| type | Integer | 题目类型（1=单选 2=多选 3=判断） |
| content | TEXT | 题目内容（支持富文本/HTML，建议不超过5000字符） |
| contentHash | String(64) | 题目内容MD5哈希（用于导入时重复检测，建议索引） |
| parse | TEXT | 题目解析（可选，支持富文本，建议不超过2000字符） |
| score | Integer | 默认分值 |
| difficulty | Integer | 难度（1=简单 2=中等 3=困难，可选） |
| correctAnswer | String | 正确答案（判断题专用，存"1"=对 "0"=错；单选/多选答案走选项表） |
| status | Integer | 状态（0=禁用 1=启用，默认1） |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |
| deleted | Boolean | 是否删除 |

**题目内容重复检测：**
```
1. 导入时：根据 bankId + contentHash 检测是否已存在相同题目
2. 处理策略（可配置）：
   - 严格模式：拒绝导入，提示"题目已存在"
   - 宽松模式：跳过重复题目，继续导入其他题目
   - 更新模式：用Excel内容覆盖已有题目
3. contentHash 字段说明：
   - 使用 MD5(content) 计算，存储为 VARCHAR(64)
   - 索引：INDEX idx_bank_hash (bank_id, contentHash)
   - 仅针对题目内容计算，不包含解析、分数等可变字段
   - 碰撞防护：MD5 理论上存在碰撞风险，重复检测时应以 bank_id + content_hash 联合判断，
     命中后再做内容字符串 equals 二次确认。如对安全性要求极高，可升级为 SHA-256
```

**题型扩展说明（V1现状 & V2规划）：**
```
当前V1版本仅支持客观题（单选/多选/判断）。
后续如需扩展主观题（填空、问答、材料分析），建议方案：
1. exam_question 增加 answer_mode 字段（1=客观题 2=主观题）
2. 主观题答案存于 correct_answer 字段（文本型），不走选项表
3. 答题记录 user_answer 改为存文本内容，评分走人工或AI批改流程
4. 本设计文档后续章节已为主观题预留字段扩展空间
```

**选项实体设计（仅单选/多选；判断题无选项记录）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| questionId | Long | 题目ID |
| optionKey | String | 选项标识（如 A/B/C/D，用于作答和展示） |
| content | String | 选项内容（支持富文本/HTML，VARCHAR(2000)） |
| isCorrect | Boolean | 是否为正确答案 |
| sort | Integer | 排序 |
| status | Integer | 状态（0=禁用 1=启用，默认1） |

**选项表唯一约束：** 同一题目下 optionKey 必须唯一（如不能有两个 "A" 选项）

**判断题特殊处理：**
```
1. 判断题不创建选项记录（ exam_question_option 表中无数据）
2. 判断题正确答案直接存储在 exam_question.correct_answer 字段："1"=对，"0"=错
3. 导入模板中：判断题的"正确答案"列填写"对"或"错"，后端映射为 1/0
4. 快照中的判断题数据结构：
   {
     "type": 3,
     "content": "危化品可以与普通物品混存。",
     "score": 2,
     "correctAnswer": ["0"],  // 0=错
     "options": {}  // 判断题无选项，前端固定渲染"对/错"
   }
5. 答题记录中：userAnswer 存储 "1" 或 "0"
```

---

## 二、试卷管理

**试卷实体设计：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| name | String | 试卷名称 |
| description | String | 试卷描述 |
| examMode | Integer | 考试模式（1=练习模式 2=正式考试） |
| totalScore | Integer | 总分 |
| passingScore | Integer | 及格分数 |
| durationMinutes | Integer | 考试时长（分钟） |
| totalTimes | Integer | 最大考试次数（0=不限制） |
| validStartTime | Date | 有效开始时间（可选） |
| validEndTime | Date | 有效结束时间（可选） |
| status | Integer | 状态（0=禁用 1=启用） |
| showAnswerMode | Integer | 答案显示模式（1=交卷后立即显示 2=考试结束后统一显示 3=不显示） |
| examPlanId | Long | 关联考试安排ID（可选，用于支持同试卷多次复用，见4.7节） |
| multiScoreStrategy | Integer | 多选评分策略（1=完全匹配得分 2=少选得半分 3=少选按比例得分） |
| totalScoreAuto | Boolean | 总分是否自动计算（默认true，根据关联题型配置自动求和，关闭则可手动设置总分） |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |
| deleted | Boolean | 是否删除 |

**试卷-题库关联实体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID |
| bankId | Long | 题库ID |
| questionType | Integer | 题型（1=单选 2=多选 3=判断） |
| questionCount | Integer | 抽取题目数量 |
| perScore | Integer | 每题分值 |

**试卷-题库关联唯一约束：** 同一试卷+同一题库+同一题型只能有一条配置记录

**试卷配置修改规则：**
```
1. 试卷一旦有 status=0（进行中）的考试记录，立即锁定以下配置：
   - 题型数量（questionCount）
   - 每题分值（perScore）
   - 考试时长（durationMinutes）
   - 多选评分策略（multiScoreStrategy）
   - 答案显示模式（showAnswerMode）
   - 题库关联配置（exam_exam_bank_question）

2. 锁定后允许修改的配置：
   - 试卷名称、描述
   - 试卷状态（启用/禁用）
   - 允许角色（但不影响已有进行中的考试）

3. 锁定解除条件：所有进行中的考试记录都已结束（status=1 或 status=2）

4. 实现方式：
   - 试卷表增加 locked_time 字段，记录锁定时间
   - 或通过检查是否存在进行中的考试记录来判断
```

**试卷总分自动计算与校验：**
```
1. 创建/修改试卷时，若 total_score_auto=true，系统自动计算总分：
   totalScore = SUM(exam_exam_bank_question.question_count * per_score)
2. 若 total_score_auto=false，允许管理员手动设置总分（用于特殊加权场景）
3. 保存前校验：自动计算的总分必须 > 0，且与手动设置值一致（若开启了自动计算）
4. 考试开始时再次校验：防止配置修改后总分与题型配置不一致
```

**试卷-允许角色关联实体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID |
| roleId | Long | 角色ID |

**试卷-允许用户关联实体（支持用户级精确授权）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID |
| userId | Long | 用户ID |
| expireTime | DateTime | 授权过期时间（可选，为空表示不过期） |

**授权校验逻辑：**
```
1. 用户进入考试 → 检查是否有该试卷的考试资格
2. 检查顺序：
   a) 检查用户级白名单（exam_exam_user），命中则允许
   b) 检查用户角色是否在试卷允许角色列表中，命中则允许
   c) 否则拒绝
3. 用户级授权优先级高于角色授权（可绕过角色限制）
```

**试卷快照实体（考试开始时生成，保证历史数据一致性）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID |
| examRecordId | Long | 考试记录ID（唯一关联） |
| snapshotData | TEXT | 快照数据（JSON格式，包含完整的题目、选项、正确答案、分值） |
|  |  | 注意：单张试卷快照可能达 50KB+，建议使用 TEXT 或 MEDIUMTEXT |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |

**快照JSON数据结构：**
```json
{
  "examId": 1,
  "questionOrder": [101, 105, 203, 102],  // 题目顺序（打乱后的顺序）
  "questions": {
    "101": {
      "type": 1,
      "content": "题目内容（富文本）",
      "score": 5,
      "optionsOrder": ["A", "C", "B", "D"],  // 选项顺序（打乱后的顺序）
      "options": {
        "A": {"optionId": 1001, "content": "选项A"},
        "C": {"optionId": 1003, "content": "选项C"},
        "B": {"optionId": 1002, "content": "选项B"},
        "D": {"optionId": 1004, "content": "选项D"}
      },
      "correctAnswer": ["A"],
      "parse": "题目解析"
    }
  }
}
```

**说明：**
- `questionOrder`：记录题目顺序，用于恢复考试时的题目展示顺序
- `optionsOrder`：记录选项打乱后的展示顺序
- `questions`：以 questionId 为 key 的 map，方便按 ID 快速查找
- 不同考生的同一道题，选项打乱后顺序不同，保证作弊难度
- 判断题在快照中无 options 和 optionsOrder 字段（前端固定渲染"对/错"）

---

## 三、考试记录与成绩

**考生答题记录实体（核心表）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examRecordId | Long | 考试记录ID |
| snapshotId | Long | 试卷快照ID（关联当时的题目快照） |
| questionId | Long | 题目ID |
| questionType | Integer | 题目类型 |
| userAnswer | String | 用户答案（JSON格式，存储 optionKey） |
| correctAnswer | String | 正确答案（JSON格式，存储 optionKey） |
| answerStatus | Integer | 答题状态（0=未答 1=已答）**注**：V1简化设计，暂不使用"暂存"状态 |
| isCorrect | Boolean | 是否正确 |
| score | Integer | 本题得分 |
| answerDuration | Integer | 答题时长（秒，用于统计分析） |
| creator | String | 答题人 |
| createTime | Date | 答题时间 |

**JSON格式说明：**
- 单选：`"userAnswer": "A"` `correctAnswer: "A"`（optionKey）
- 多选：`"userAnswer": ["A","B"]` `correctAnswer": ["A","B"]`（optionKey）
- 判断：`"userAnswer": "1"` `correctAnswer": "0"`（1=对，0=错）

**考试记录实体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID |
| userId | Long | 考生ID |
| examMode | Integer | 考试模式（1=练习模式 2=正式考试） |
| totalScore | Integer | 考试成绩 |
| correctCount | Integer | 正确题数 |
| totalCount | Integer | 总题数 |
| status | Integer | 状态（0=进行中 1=已提交 2=已结束） |
| submitType | Integer | 提交方式（0=未提交 1=主动提交 2=超时自动提交 3=管理员/系统强制提交） |
| cheatingFlag | Boolean | 是否疑似作弊（false=正常 true=疑似作弊） |
| startTime | Date | 开始时间 |
| endTime | Date | 预计结束时间（startTime + durationMinutes，用于超时扫描索引） |
| submitTime | Date | 提交时间 |
| ip | String | 答题IP |
| userAgent | String | 浏览器信息 |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |
| deleted | Boolean | 是否删除 |

---

## 四、业务流程

### 4.1 考试流程

```
1. 考生进入考试列表 → 选择试卷
2. 系统检查：
   ├── 角色权限（是否允许参加）
   ├── 考试次数（是否已达上限）
   ├── 有效时间（是否在有效期内）
   └── 启用状态（试卷是否启用）
3. 检查通过 → 创建考试记录（status=进行中, endTime=now+durationMinutes）
4. 考生随机抽题 → 开始答题
5. 答题过程：
   ├── 考生选择答案 → 保存答题记录（含本题答题时长，用于统计分析）
   └── 倒计时计时
6. 提交方式：
   ├── 主动提交：考生点击提交按钮
   └── 超时提交：倒计时结束自动提交
7. 提交后 → 计算成绩 → 更新考试记录 → 返回成绩

**断线续考边界处理：**
- 调用 /student-exam/recover 时，后端检查 endTime 是否已过期
- 如已超时：返回 status=已结束，前端引导至成绩页面
- 如未超时：返回完整答题进度和剩余时间
```

### 4.2 随机抽题算法

```
1. 根据试卷配置的"试卷-题库关联"表
2. 对每个题型分别从对应题库中随机抽取指定数量
   - 边界处理：如题库实际数量 < 配置抽取数量
     ├── 策略A（严格模式）：抛异常"题库题目不足，无法组卷"
     └── 策略B（宽松模式）：取实际全部数量（min(配置数量, 实际数量)）
3. 题目随机打乱顺序
4. 返回题目列表（不含正确答案）
5. 正确答案单独存储，用于后续评分
```

**抽题数量不足处理伪代码：**
```java
public List<Question> randomQuestions(Long examId) {
    List<Question> result = new ArrayList<>();
    List<ExamBankQuestion> configs = examBankQuestionMapper.selectByExamId(examId);
    for (ExamBankQuestion config : configs) {
        int availableCount = questionMapper.countByBankAndType(config.getBankId(), config.getQuestionType());
        int needCount = config.getQuestionCount();
        if (availableCount < needCount) {
            // 策略B：取实际可用数量（如需严格模式则抛异常）
            needCount = availableCount;
        }
        List<Question> questions = questionMapper.selectRandomByBankAndType(
            config.getBankId(), config.getQuestionType(), needCount
        );
        result.addAll(questions);
    }
    Collections.shuffle(result);
    return result;
}
```

### 4.3 自动评分算法

```
1. 遍历考生答题记录（基于试卷快照，不受题库后续修改影响）
2. 未作答（userAnswer 为空/null）直接得零分
3. 单选题/判断题：直接比对 optionKey 是否相等
4. 多选题：根据试卷配置的 multiScoreStrategy 进行评分
   ├── 完全匹配得分：userAnswer 与 correctAnswer 完全一致得满分
   ├── 少选得半分：少选（选项均为正确答案子集）得一半分，多选/错选零分
   └── 少选按比例得分：每正确选择一个选项得 perOptionScore 分
5. 累加得分
6. 更新考试记录的总分和状态
```

**多选题评分伪代码（修复边界 case）：**
```java
public int calculateMultiScore(List<String> userAnswer, List<String> correctAnswer,
                               Integer strategy, int fullScore) {
    // 1. 处理未作答或空答案
    if (userAnswer == null || userAnswer.isEmpty()) {
        return 0; // 未作答得零分
    }

    // 2. 检测错选（选了不在正确答案中的选项）
    boolean hasWrongOption = !correctAnswer.containsAll(userAnswer);
    if (hasWrongOption) {
        return 0; // 错选直接零分
    }

    // 3. 正好完全匹配
    if (userAnswer.equals(correctAnswer)) {
        return fullScore;
    }

    // 4. 少选情况
    if (userAnswer.size() < correctAnswer.size()) {
        if (strategy == 1) { // 完全匹配策略：少选零分
            return 0;
        } else if (strategy == 2) { // 少选得半分
            return fullScore / 2;
        } else if (strategy == 3) { // 按比例得分
            int perScore = fullScore / correctAnswer.size();
            return perScore * userAnswer.size();
        }
    }

    // 5. 多选情况（userAnswer.size() > correctAnswer.size()）
    if (strategy == 1) {
        return 0; // 完全匹配策略：多选零分
    } else if (strategy == 2) {
        return 0; // 少选得半分策略：多选零分
    } else if (strategy == 3) {
        return fullScore; // 已排除错选，多选的都是正确选项，得满分
    }

    return 0;
}
```

**多选题评分示例（满分 4 分，正确答案 ["A","B","C"]）：**

| 用户答案 | 完全匹配 | 少选得半分 | 按比例得分（每正确项1分） |
|----------|----------|------------|---------------------------|
| ["A","B","C"] | 4分 | 4分 | 4分 |
| ["A","B"] | 0分 | 2分 | 2分 |
| ["A","B","D"] | 0分 | 0分 | 0分 |  （D为错误选项，含错选一律零分） |
| ["A","B","C","D"] | 0分 | 0分 | 0分 |  （D为错误选项，含错选一律零分） |

### 4.4 练习模式 vs 正式考试差异

| 功能维度 | 练习模式（examMode=1） | 正式考试（examMode=2） |
|----------|----------------------|----------------------|
| 次数限制 | 不限次数（totalTimes 无效） | 受 totalTimes 限制，0=不限 |
| 时间限制 | 显示用时但不强制倒计时 | 严格倒计时，超时自动提交 |
| 答案显示 | 交卷后立即显示答案+解析 | 受 showAnswerMode 控制 |
| 成绩记录 | 记录答题记录但不计入统计 | 计入成绩统计、排名 |
| 及格处理 | 可无限反复练习 | 完成后是否允许重考由业务决定 |
| 防作弊 | 不启用 | 启用切屏检测等 |
| 重新进入同一试卷 | **重新抽题**（每次进入都是新试卷） | **恢复进度**（断线续考） |
| 答错收录错题本 | **交卷后立即收录**（即时反馈） | **整卷提交后统一收录** |
| 切屏检测 | 不记录 | 记录切屏次数，超限强制交卷 |
| 数据落表 | 复用 exam_record + exam_answer 表，exam_mode=1 | 复用 exam_record + exam_answer 表，exam_mode=2 |

**练习模式详细规则：**
```
1. 进入练习：每次进入都重新随机抽题（题目顺序、选项顺序均打乱）
2. 答题过程：交卷后立即显示本题正确答案和解析（即时反馈）
3. 错题收录：答错即收录到错题本，无需整卷结束
4. 时间管理：无强制倒计时，显示"已用时间"供参考
5. 重复练习：支持同一试卷反复练习，每次都是新抽题
6. 答题记录：记录答题历史但不参与成绩统计和排名
7. 数据存储：练习模式复用 exam_record + exam_answer 表，通过 exam_mode=1 区分。
   练习记录查询时需主动过滤 exam_mode=2（正式考试），避免污染成绩统计
```

### 4.4.1 答案显示模式语义补充
```
showAnswerMode=1（交卷后立即显示）：
  - 考生提交后，立即可查看成绩、正确答案、解析
  - 适用于练习模式或即时反馈型考试

showAnswerMode=2（考试结束后统一显示）：
  - V1无考试安排时："结束后"定义为该试卷的 valid_end_time 已过期，
    或管理员手动触发"公布答案"操作
  - V2有考试安排时（见4.7）："结束后"定义为 exam_plan 的 valid_end_time 已过期，
    或管理员手动触发"公布答案"操作
  - 未公布前，考生只能看到"已提交"状态，看不到具体分数和答案
  - 适用于需要统一时间开榜的正式考试

showAnswerMode=3（不显示）：
  - 任何情况下考生端不显示正确答案和解析
  - 管理端仍可查看统计数据和考生答题详情
```

### 4.5 租户隔离方案

所有考试模块实体统一添加 `tenantId` 字段：

| 实体 | tenantId 说明 |
|------|---------------|
| 题库 ExamBank | 标识属于哪个租户 |
| 题目 Question | 标识属于哪个租户 |
| 选项 QuestionOption | 标识属于哪个租户 |
| 试卷 Exam | 标识属于哪个租户 |
| 试卷-题库关联 ExamBankQuestion | 标识属于哪个租户 |
| 试卷-允许角色关联 ExamRole | 标识属于哪个租户 |
| 考试记录 ExamRecord | 标识属于哪个租户 |
| 答题记录 ExamAnswer | 标识属于哪个租户 |

**权限校验逻辑：**
```
1. 用户登录 → 获取用户的 tenantId 和 角色列表
2. 查看试卷列表 → 只返回该 tenantId 下的试卷
3. 参加考试 → 检查试卷的允许角色列表是否与用户角色有交集
4. 答题记录 → 只查询该 tenantId 下的记录
```

### 4.7 考试安排与试卷复用（可选扩展）

**问题背景：**
```
当前设计中 "试卷(Exam)" = "考试活动"，如果同一套试题需要：
- 3月给A部门考，5月给B部门考
- 上半年和下半年各组织一次
则需要复制多份完全相同的试卷配置，维护成本高
```

**解决方案（按需引入 exam_plan 表）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 关联试卷ID |
| name | String | 考试安排名称（如"2026上半年危化品考核"） |
| validStartTime | DateTime | 允许进入考试的开始时间 |
| validEndTime | DateTime | 允许进入考试的结束时间 |
| showAnswerTime | DateTime | 答案公布时间（showAnswerMode=2时使用） |
| status | Integer | 状态（0=未开始 1=进行中 2=已结束） |

**引入后的变化：**
```
1. 试卷(Exam) 退化为"试题模板"，只配置题型、分值、策略
2. 考试安排(ExamPlan) 承载时间窗口、参考人员、答案公布时间
3. 考试记录(ExamRecord) 增加 exam_plan_id 字段，排名按 exam_plan_id 维度统计
4. 若不引入此表，则文档中所有 exam_plan_id 逻辑均回退到 exam_id
```

> **V1建议：** 暂不引入 exam_plan，保持简单。如业务确需复用试卷，再按此扩展。

### 4.6 并发控制与分布式锁

**需要加锁的关键场景：**

| 场景 | 锁Key | 说明 |
|------|-------|------|
| 开始考试 | `exam:lock:start:{userId}:{examId}` | 防止并发导致超次考试 |
| 提交考试 | `exam:lock:submit:{recordId}` | 防止重复提交、重复计分 |
| 更新题库 | `exam:lock:bank:{bankId}` | 防止并发修改题目导致数据不一致 |

**伪代码示例（开始考试）：**
```java
public ExamRecord startExam(Long userId, Long examId) {
    String lockKey = "exam:lock:start:" + userId + ":" + examId;
    boolean locked = redisLock.tryLock(lockKey, 10, TimeUnit.SECONDS);
    if (!locked) throw new BizException("操作过于频繁，请稍后重试");
    try {
        // 1. 校验是否有进行中的考试
        int ongoingCount = examRecordMapper.countOngoingByUser(userId);
        if (ongoingCount > 0) {
            throw new BizException("您有进行中的考试，请先完成");
        }
        // 2. 校验考试次数（仅统计正式考试且已完成/已提交的记录，不包含进行中和练习模式）
        int usedTimes = examRecordMapper.countFinishedByUserAndExam(userId, examId, 2); // exam_mode=2
        if (exam.getTotalTimes() > 0 && usedTimes >= exam.getTotalTimes()) {
            throw new BizException("考试次数已达上限");
        }
        // 3. 创建考试记录
        ExamRecord record = createExamRecord(userId, examId);
        // 3. 生成试卷快照
        ExamPaperSnapshot snapshot = generateSnapshot(record.getId(), examId);
        return record;
    } finally {
        redisLock.unlock(lockKey);
    }
}
```

---

## 五、API 接口设计

**接口版本控制说明：**
```
1. 当前版本：v1（如 /exam/v1/bank/create）
2. 版本升级规则：
   - 重大架构调整时升级版本号
   - 向下兼容的字段增删不影响版本
   - 不兼容的变更需要新版本
3. 建议长期维护接口采用版本控制
4. 如系统仅供内部使用，可暂不使用版本控制
```

### 5.1 题库管理接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam-bank/create | POST | 创建题库 |
| /exam-bank/update | PUT | 更新题库 |
| /exam-bank/delete | DELETE | 删除题库 |
| /exam-bank/page | GET | 分页查询题库 |
| /exam-bank/get | GET | 获取题库详情 |

### 5.2 题目管理接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /question/create | POST | 创建题目 |
| /question/update | PUT | 更新题目 |
| /question/delete | DELETE | 删除题目（被试卷引用时禁止硬删） |
| /question/page | GET | 分页查询题目 |
| /question/get | GET | 获取题目详情 |
| /question/simple-list | GET | 获取某题库下的题目精简列表 |

**题目删除/修改限制：**
```
1. 删除前检查：查询 exam_exam_bank_question 表，若该题目所属题库已被某试卷引用
   ├── 若试卷无进行中/已提交的考试记录：允许删除，但提示"该题目已被N张试卷引用，删除后不影响历史考试"
   └── 若试卷存在进行中/已提交的考试记录：禁止删除，返回"题目正在被使用中，无法删除"
2. 修改题目：始终允许修改（包括被引用的题目），因为历史考试使用快照，不受修改影响
3. 推荐做法：题目采用软删（deleted=1），保留数据完整性，已被引用的题目仅标记删除
```

### 5.3 试卷管理接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam/create | POST | 创建试卷 |
| /exam/update | PUT | 更新试卷 |
| /exam/delete | DELETE | 删除试卷 |
| /exam/page | GET | 分页查询试卷 |
| /exam/get | GET | 获取试卷详情 |
| /exam/update-status | PUT | 更新试卷状态 |
| /exam/config-roles | PUT | 配置允许角色 |

### 5.4 考试接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam-record/page | GET | 分页查询考试记录（管理端） |
| /exam-record/get | GET | 获取考试记录详情 |
| /exam-record/statistics | GET | 考试成绩统计（管理端） |
| /exam-record/export | GET | 导出考试成绩Excel |
| /student-exam/start | POST | 学生开始考试 |
| /student-exam/submit | PUT | 提交考试答案 |
| /student-exam/undo | PUT | 撤销提交（**V1暂不支持**，提交后不可撤回） |
| /student-exam/save | PUT | 自动保存/暂存答案（心跳，含每题答题时长上报） |
| /student-exam/recover | GET | 恢复考试进度（断线续考，考试已超时则返回状态并引导查看成绩） |
| /student-exam/my-page | GET | 我的考试记录（学生端） |
| /student-exam/detail | GET | 我的答题详情（学生端） |
| /student-exam/summary | GET | 我的成绩单（按时间范围查询，支持导出） |
| /student-exam/trend | GET | 我的成绩趋势（近N次考试的分数变化曲线） |

### 5.5 题目批量操作接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /question/template-download | GET | 下载题目导入Excel模板 |
| /question/import | POST | Excel批量导入题目（**同步**，适合小量导入） |
| /question/import-async | POST | Excel批量导入题目（**异步**，适合大量导入，返回任务ID） |
| /question/import-progress | GET | 查询异步导入进度（传入任务ID） |
| /question/export | GET | 导出题目Excel |
| /question/upload-image | POST | 题目图片上传（返回URL） |
| /question/batch-delete | DELETE | 批量删除题目 |
| /question/batch-update | PUT | 批量更新题目（如难度、分数） |

**异步导入说明：**
```
1. 前端调用 /question/import-async 上传Excel，返回任务ID
2. 后端创建导入任务（exam_import_task 表），立即返回任务ID
3. 前端每2秒轮询 /question/import-progress 查询进度
4. 导入完成后，返回成功/失败数量和错误详情
5. 任务超时时间：30分钟（超时自动标记失败）
6. 错误处理：跳过格式错误的行，保留错误清单，最后统一返回
```

**导入Excel模板字段：**
| 字段 | 必填 | 说明 |
|------|------|------|
| 题库名称 | 是 | 关联题库 |
| 题目类型 | 是 | 单选/多选/判断 |
| 题目内容 | 是 | 支持富文本（HTML标签） |
| 选项A | 条件必填 | 单选/多选必填 |
| 选项B | 条件必填 | 单选/多选必填 |
| 选项C | 条件必填 | 单选/多选必填 |
| 选项D | 条件必填 | 单选/多选必填 |
| 选项E | 否 | 多选可选 |
| 选项F | 否 | 多选可选 |
| 正确答案 | 是 | 单选填A/B/C/D，多选填AB/AC等，判断填"对"或"错"（也可填1/0，后端自动映射） |
| 分值 | 否 | 默认1分 |
| 解析 | 否 | 题目解析 |
| 难度 | 否 | 简单/中等/困难 |

### 5.6 试卷批量操作接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam/batch-delete | DELETE | 批量删除试卷 |
| /exam/batch-update-status | PUT | 批量启用/禁用试卷 |

### 5.7 考试监控接口（管理端）

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam-monitor/live-list | GET | 实时参考人数监控 |
| /exam-monitor/suspicious-list | GET | 可疑行为列表（切屏超限、答题时间异常等） |
| /exam-monitor/behavior/{recordId} | GET | 某考生的完整行为日志 |
| /exam-monitor/force-submit | POST | 管理员强制提交某考生的试卷 |

### 5.8 操作审计日志接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam-audit/log/page | GET | 操作审计日志分页查询 |
| /exam-audit/log/create | POST | 记录操作日志（系统自动调用） |

### 5.9 通知消息接口

| 接口 | 方法 | 说明 |
|------|------|------|
| /exam-notification/page | GET | 通知消息分页查询（学生端） |
| /exam-notification/unread-count | GET | 未读消息数量（用于红点提醒） |
| /exam-notification/mark-read | PUT | 标记已读（支持单条和批量） |
| /exam-notification/mark-all-read | PUT | 标记全部已读 |

---

## 六、前端页面设计

### 6.1 管理端页面（管理员角色）

| 页面路径 | 说明 |
|----------|------|
| /exam/bank/index | 题库管理 |
| /exam/bank/detail | 题库详情（含题目管理） |
| /exam/question/index | 题目管理 |
| /exam/question/detail | 题目详情（创建/编辑） |
| /exam/exam/index | 试卷管理 |
| /exam/exam/detail | 试卷详情（创建/编辑） |
| /exam/record/index | 考试记录列表 |
| /exam/record/detail | 考试记录详情（查看答题情况） |

### 6.2 考试入口（所有登录用户）

| 页面路径 | 说明 |
|----------|------|
| /exam/user/index | 我的考试（可参加的考试列表） |
| /exam/user/do/[id] | 在线答题 |
| /exam/user/result/[id] | 考试成绩 |
| /exam/user/history | 我的考试记录 |

### 6.3 权限控制

```
├── 管理员（拥有 exam:bank / exam:question / exam:exam 权限）
│   └── 可访问题库管理、题目管理、试卷管理、考试记录、成绩统计
│
└── 普通用户（无上述权限）
    └── 只能访问 /exam/user/* 菜单、错题本
```

### 6.4 新增管理端页面

| 页面路径 | 说明 |
|----------|------|
| /exam/statistics/index | 成绩统计看板（及格率、平均分、题目正确率） |
| /exam/statistics/detail | 单张试卷的详细统计 |
| /exam/question/import | 题目批量导入页面 |

### 6.5 新增学生端页面

| 页面路径 | 说明 |
|----------|------|
| /exam/user/wrong-book | 错题本（收录所有做错的题目） |
| /exam/user/favorite | 收藏夹（收藏重要题目） |
| /exam/user/exam-notice | 考试须知/确认页面 |

### 6.6 答题页面交互增强

**断线续考机制：**
```
1. 考生进入答题页面 → 先调用 /student-exam/recover 获取已保存进度
2. 答题过程中 → 每30秒自动调用 /student-exam/save 保存当前答案
3. 本地同时存储到 localStorage（Key 包含 userId，避免多账号共用浏览器时数据串台） → 网络异常时先从本地恢复
4. 提交时 → 合并服务端暂存答案 + 本地答案，以最新为准

**答题时长埋点：**
- 前端记录考生进入每道题的时刻（startTime）和提交答案的时刻（endTime）
- 调用 /student-exam/save 时，通过 questionTimes 参数上报每题耗时
- 后端存储到 ExamAnswer 表（需要新增 answerDuration 字段），用于统计分析
```

**数据库事务边界：**
```
以下操作必须保证原子性（@Transactional）：
1. 开始考试：创建 exam_record → 生成 exam_paper_snapshot → 批量插入 exam_answer（初始状态）
2. 提交考试：更新 exam_record（成绩/状态） → 批量更新 exam_answer（答案/得分）
3. 强制提交：更新 exam_record → 插入 exam_behavior_log

事务粒度控制：
- 开始考试时批量插入 exam_answer 的数量 = 题目数（最多200题），在单事务可控范围内
- 若题目数量极大（>500题），建议分批次插入，但需保证最终一致性
- 练习模式与正式考试共用同一套事务逻辑，通过 exam_mode 字段区分业务规则
```

### 6.7 答题页面交互

```
┌─────────────────────────────────────────┐
│  试卷名称：危化品安全知识测试    剩余时间：45:32 │
├─────────────────────────────────────────┤
│  一、单项选择题（共10题，每题5分）              │
│  ┌─────────────────────────────────┐    │
│  │ 1. 下列哪种物质属于危化品？      │    │
│  │   ○ A. 食盐    ○ B. 酒精        │    │
│  │   ○ C. 水      ○ D. 木材        │    │
│  └─────────────────────────────────┘    │
│                                         │
│  二、多项选择题（共5题，每题6分）              │
│  ┌─────────────────────────────────┐    │
│  │ 2. 危化品存储需要满足哪些条件？  │    │
│  │   ☑ A. 通风  ☑ B. 防火         │    │
│  │   ☑ C. 防潮  □ D. 防雷           │    │
│  └─────────────────────────────────┘    │
│                                         │
│  三、判断题（共10题，每题2分）                │
│  ┌─────────────────────────────────┐    │
│  │ 3. 危化品可以与普通物品混存。    │    │
│  │   ○ 对    ○ 错                  │    │
│  └─────────────────────────────────┘    │
├─────────────────────────────────────────┤
│           [ 交卷并查看成绩 ]               │
└─────────────────────────────────────────┘
```

---

## 七、错误处理策略

### 7.1 考试开始失败

| 错误场景 | 错误码 | 提示信息 |
|----------|--------|----------|
| 试卷不存在 | 1-001 | 试卷不存在 |
| 试卷已禁用 | 1-002 | 该试卷已禁用 |
| 不在有效期 | 1-003 | 不在考试有效期内 |
| 次数已达上限 | 1-004 | 您已达到该试卷的最大考试次数 |
| 无参考权限 | 1-005 | 您没有权限参加此考试 |
| 存在进行中的考试 | 1-006 | 您有进行中的考试，请先完成 |

### 7.2 答题异常

| 错误场景 | 错误码 | 提示信息 |
|----------|--------|----------|
| 考试记录不存在 | 2-001 | 考试记录不存在 |
| 考试已超时 | 2-002 | 考试已超时，已自动提交（如继续答题请以新记录开始） |
| 考试已提交 | 2-003 | 考试已提交，不能重复提交 |
| 题目不在考试范围内 | 2-004 | 题目不在本次考试范围内 |
| 操作过于频繁 | 2-005 | 请勿频繁操作，请稍后重试 |
| 考试已结束 | 2-006 | 该考试已结束，无法继续答题 |

### 7.3 异常处理流程

```
1. 考试开始时校验 → 不通过则返回错误码和提示
2. 答题过程中校验 → 保存失败可重试（本地缓存兜底）
3. 提交时再次校验 → 分布式锁防止重复提交
4. 超时自动提交 → 后台定时任务扫描超时记录
5. 断线恢复 → 优先从服务端恢复，服务端无数据则从本地 localStorage 恢复
```

### 7.4 考试暂停与不可抗力兜底

**考试暂停策略：**
```
1. 正式考试原则上不允许主动暂停
2. 考生因电脑死机、断电、浏览器崩溃等不可抗力中断：
   - 中断后 5 分钟内恢复：正常断线续考（end_time 不变）
   - 中断超过 5 分钟或已超时：考试自动结束，按已答题目提交评分
   - 特殊情况：考生可向管理员申诉，管理员可重置该考生的考试记录（删除原记录，允许重考）
3. 练习模式：无此限制，每次进入都是新试卷
```

---

## 八、考试监控与防作弊

### 8.1 防作弊策略

| 策略 | 实现方式 | 适用场景 |
|------|----------|----------|
| 切屏检测 | 前端监听 `visibilitychange` 事件，记录切出次数（桌面端有效；移动端iOS Safari等可能不支持，需辅以 blur/focus 时间差检测） | 正式考试 |

**移动端切屏检测方案：**
```
1. visibilitychange 事件（移动端支持较好）：
   - 监听 document.visibilityState 变化
   - 切换到 background 时记录切屏

2. blur/focus 时间差检测（兜底方案）：
   - 监听 window blur 事件（切换APP/接电话）
   - 记录离开时间，focus 时计算离开时长
   - 离开时长超过 5秒 算作一次切屏

3. 推荐浏览器提示：
   - 考试开始前提示"推荐使用 Chrome/Edge 浏览器"
   - iOS Safari 用户提示可能无法完整检测切屏

4. 实现伪代码：
```javascript
let leaveTime = null;
window.addEventListener('blur', () => {
    leaveTime = Date.now();
});
window.addEventListener('focus', () => {
    if (leaveTime && Date.now() - leaveTime > 5000) {
        reportScreenSwitch(); // 上报切屏
    }
    leaveTime = null;
});
```
```
| 单设备限制 | 同一时间同一账号只能有一个进行中的考试记录 | 正式考试 |
| 选项乱序 | 不同考生的同一道题选项顺序随机打乱 | 正式考试 |
| 题目乱序 | 不同考生的题目顺序随机打乱（已支持） | 正式考试 |
| IP 记录 | 记录考试 IP，管理员可查看 | 所有考试 |
| 答题时间异常检测 | 某道题答题时间过短（如<3秒）标记异常 | 正式考试 |

### 8.2 监控数据记录

**考试行为日志实体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examRecordId | Long | 考试记录ID |
| userId | Long | 用户ID |
| actionType | Integer | 行为类型（1=切屏 2=复制粘贴 3=窗口失焦 4=异常提交） |
| actionDesc | String | 行为描述 |
| createTime | Date | 发生时间 |
| deleted | Boolean | 是否删除 |

**行为日志索引优化：** 管理端通常按考试记录查询行为日志并按时间排序
```sql
-- 建议使用联合索引替代单列 create_time 索引
INDEX idx_record_time (exam_record_id, create_time)
```

### 8.3 异常处理规则

**防作弊规则配置表（exam_cheating_rule）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| examId | Long | 试卷ID（为空表示默认规则） |
| ruleType | Integer | 规则类型（1=切屏次数 2=答题时间异常 3=复制粘贴） |
| warnThreshold | Integer | 警告阈值 |
| forceSubmitThreshold | Integer | 强制提交阈值（0表示不强制提交） |
| enabled | Boolean | 是否启用 |
| deleted | Boolean | 是否删除（软删） |

**默认规则（examId 为空时使用）：**
```
切屏次数 >= 3 次 → 记录警告，管理员可见
切屏次数 >= 5 次 → 强制提交试卷，标记为"疑似作弊"
复制粘贴题目内容 → 记录行为日志（前端拦截+后端记录）
答题时间异常（如平均答题时间 < 3秒/题）→ 标记为"疑似作弊"
```

**规则匹配逻辑：**
```
1. 优先使用试卷专属规则（examId 精确匹配）
2. 无专属规则则使用租户默认规则
3. 规则可针对单个试卷定制（如重要考试更严格）
```

**防作弊规则表设计优化建议：**
```
当前 exam_cheating_rule 设计对不同 rule_type 的阈值语义不一致，建议优化：

方案A（推荐）：规则配置改为JSON格式存储参数
  rule_config 字段存储 JSON，如：
  - 切屏规则：{"unit": "times", "warn": 3, "forceSubmit": 5}
  - 答题时间异常：{"unit": "seconds_per_question", "threshold": 3}
  - 复制粘贴：{"action": "log_only", "forceSubmit": 0}

方案B：按规则类型分字段存储
  - screen_switch_warn / screen_switch_force
  - min_answer_duration_seconds
  - copy_paste_action (log/block)
```

---

## 九、成绩统计与分析

### 9.1 统计维度

**单张试卷统计：**

| 统计项 | 说明 |
|--------|------|
| 参考人数 | 实际参加考试的人数 |
| 及格人数/及格率 | 达到 passingScore 的人数和比例 |
| 平均分 | 所有考生总分的平均值 |
| 最高分/最低分 | 成绩区间 |
| 分数段分布 | 0-59, 60-69, 70-79, 80-89, 90-100 |
| 平均用时 | 所有考生的平均答题时长 |

**每道题统计（用于评估题目质量）：**

| 统计项 | 说明 |
|--------|------|
| 正确率 | 答对该题人数 / 总答题人数 |
| 选择分布 | 每个选项的选择人数和比例 |
| 平均用时 | 考生在该题上的平均耗时 |

### 9.2 成绩排名

**排名维度（需业务确认）：**
```
方案A（按试卷全局排名，默认）：
  - 同一 exam_id 下所有正式考试记录（exam_mode=2, status=1/2）参与排名
  - 若同一考生有多次记录，取最高分参与排名，或每条记录独立排名

方案B（按考试批次排名，需引入 exam_plan）：
  - 同一 exam_plan_id 下的考生参与排名
  - 适用于同试卷多批次考试的场景
```

**排名规则：**
```
1. 按总分降序排列
2. 总分相同 → 按用时升序排列（用时短排名靠前）
3. 支持查询"我的排名"和"前N名"
```

**注意：** 排名功能建议异步计算或缓存，避免每次查询都全表排序。
- 建议 Redis Sorted Set 缓存实时排名（ZADD score:exam:{examId} {totalScore*10000 + (9999-duration)} {userId}）
- 或定时任务每5分钟计算一次排名写入缓存

---

## 十、错题本、收藏夹与学习复盘

### 10.1 错题本实体设计

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| userId | Long | 用户ID |
| questionId | Long | 题目ID |
| bankId | Long | 题库ID |
| wrongCount | Integer | 做错次数 |
| lastWrongTime | Date | 最近一次做错时间 |
| autoRemove | Boolean | 练习模式答对后是否自动移除（默认true=自动移除） |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |

### 10.2 收藏夹实体设计

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| userId | Long | 用户ID |
| questionId | Long | 题目ID |
| bankId | Long | 题库ID |
| note | String | 收藏备注（可选） |
| creator | String | 创建者 |
| createTime | Date | 创建时间 |
| updater | String | 更新者 |
| updateTime | Date | 更新时间 |

### 10.3 错题本与收藏夹功能

**收录规则：**
```
1. 自动收录时机：答题提交时，根据 ExamAnswer.isCorrect 字段判断
   - 练习模式：每次作答提交后立即判断，答错即收录
   - 正式考试：整卷提交后统一判断，答错即收录
2. 同一题目多次答错：更新 wrongCount +1 和 lastWrongTime
3. 答对后处理：
   - 练习模式：答对后自动移出错题本（或设置开关让用户选择是否保留）
   - 正式考试：答对后不动，需要用户手动移除
4. 自动移除规则（可配置）：
   - 默认：练习模式下答对自动移除
   - 关闭：用户可设置"答对后不自动移除"
```

**功能说明：**
```
1. 反复练习：支持单独练习错题，答对后可选移除或保留
2. 错题统计：查看各题库的错题分布
3. 收藏夹：用户可手动收藏重要题目（独立于错题本）
```

**题目引用一致性处理：**
- 错题本和收藏夹中的 `questionId` 指向题目表
- 题目被删除时，显示"题目已下架"而非直接报错
- 题目被修改时，显示最新内容（错题本用于学习，应看到最新版题目）
- **建议**：在展示时检查题目是否存在，不存在则提示用户

---

## 十一、消息通知机制

### 11.1 通知场景

| 场景 | 触发时机 | 通知方式 |
|------|----------|----------|
| 考试开始提醒 | 考试开始前15分钟 | 系统消息/站内信 |
| 考试即将超时 | 剩余时间 <= 5分钟 | 页面弹窗提醒 |
| 成绩发布 | 交卷后（根据 showAnswerMode） | 系统消息 |
| 考试被强制提交 | 切屏超限或超时 | 页面提示 + 系统消息 |

### 11.2 通知实体设计

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| userId | Long | 接收用户ID |
| examId | Long | 关联试卷ID（可选，用于成绩通知等场景） |
| examRecordId | Long | 关联考试记录ID（可选，用于跳转查看成绩） |
| title | String | 通知标题 |
| content | String | 通知内容 |
| type | Integer | 类型（1=考试提醒 2=成绩通知 3=系统公告） |
| readStatus | Integer | 已读状态（0=未读 1=已读） |
| createTime | Date | 创建时间 |
| updateTime | Date | 更新时间（阅读后更新 readStatus 时使用） |

**通知触达方式：**
```
1. 站内信 + 未读红点：前端每30秒轮询 /exam-notification/unread-count 接口
2. 实时推送（可选增强）：考试开始提醒、超时警告等实时性高的通知，通过 WebSocket/SSE 推送
3. 前端收到推送后，刷新消息列表并显示 Toast 提示
4. 若用户离线，消息持久化到 exam_notification 表，登录后拉取未读列表
```

**通知跳转逻辑：**
```
1. 考试提醒（type=1）：点击跳转到 /exam/user/do/{examId}
2. 成绩通知（type=2）：点击跳转到 /exam/user/result/{examRecordId}
3. 系统公告（type=3）：仅展示内容，无需跳转
```

### 11.3 操作审计日志实体

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| tenantId | Long | 租户ID |
| operatorId | Long | 操作人ID |
| operatorName | String | 操作人名称 |
| actionType | Integer | 操作类型（1=创建 2=修改 3=删除 4=发布/下架 5=手动调整成绩 6=批量导入 7=批量删除 8=批量启用/禁用） |
| entityType | String | 实体类型（ExamBank/Question/Exam/ExamRecord等） |
| entityId | Long | 实体ID |
| entityName | String | 实体名称（如试卷名称） |
| beforeValue | String | 修改前的值（JSON格式，用于重要字段变更） |
| afterValue | String | 修改后的值（JSON格式） |
| ip | String | 操作IP |
| userAgent | String | 浏览器信息 |
| createTime | Date | 操作时间 |

**审计日志记录场景：**
```
1. 试卷创建/修改/删除
2. 试卷发布/下架
3. 成绩手动调整
4. 考生考试资格变更
5. 题库/题目批量导入/导出
```

**审计日志存储优化建议：**
- `before_value` / `after_value` 当前存储全量实体 JSON，数据膨胀快
- **建议**：只记录变更字段的 diff，如 `{"totalScore": {"old": 100, "new": 120}}`
- 可节省 50%+ 存储空间，且更便于追溯具体变更点

---

## 十二、测试策略

### 12.1 单元测试范围

| 模块 | 测试类 | 测试内容 |
|------|--------|----------|
| 题库管理 | ExamBankServiceTest | 创建/更新/删除/查询 |
| 题目管理 | QuestionServiceTest | CRUD、级联删除、Excel导入校验 |
| 试卷管理 | ExamServiceTest | 创建/配置角色/状态管理 |
| 考试服务 | ExamServiceStartTest | 开始考试权限校验、并发控制 |
| 评分服务 | ScoreCalculateTest | 单选/多选/判断评分（含部分得分策略） |
| 随机抽题 | RandomQuestionTest | 抽题数量/去重/顺序/选项乱序 |
| 快照服务 | SnapshotServiceTest | 快照生成、快照评分一致性 |

### 12.2 集成测试场景

| 场景 | 测试内容 |
|------|----------|
| 完整考试流程 | 开始→答题→提交→评分→查看成绩 |
| 角色权限控制 | 有权限用户能考、无权限用户被拒 |
| 考试次数限制 | 达上限后无法开始新考试 |
| 超时自动提交 | 倒计时结束自动提交 |
| 练习模式 | 无时间限制、可多次参加、显示解析 |
| 断线续考 | 刷新页面/关闭浏览器后恢复进度 |
| 并发测试 | 50人同时开始考试、同时提交 |
| 题库修改后 | 历史考试记录不受影响的快照验证 |

---

## 十三、技术实现要点

### 13.1 数据库设计建议

**核心实体关系（ER逻辑）：**
```
ExamBank(题库) 1:N Question(题目)
Question(题目) 1:N QuestionOption(选项)  【判断题无选项】

Exam(试卷) 1:N ExamBankQuestion(试卷-题库-题型关联配置)
Exam(试卷) 1:N ExamRole(允许角色) / 1:N ExamUser(允许用户)

ExamRecord(考试记录) 1:1 ExamPaperSnapshot(试卷快照)
ExamRecord(考试记录) 1:N ExamAnswer(答题记录)
ExamRecord(考试记录) 1:N ExamBehaviorLog(行为日志)

User(用户) 1:N ExamRecord(考试记录)
User(用户) N:M Question(错题本/收藏夹)
```

```
1. 答题记录的 user_answer 和 correct_answer 使用 JSON 字段
   - MySQL 5.7+ 原生支持
   - Java 端使用 String 存储，JSON 解析由应用层处理

2. 考试记录索引
   - INDEX idx_exam_user (exam_id, user_id)
   - INDEX idx_user_status (user_id, status)
   - INDEX idx_status_end_time (status, end_time)  -- 用于超时扫描（核心索引）

3. 题目表索引
   - INDEX idx_bank_type (bank_id, type)

**题目数量限制：**
```
1. 单张试卷题目总数上限：200题（防止快照数据过大）
2. 单个题库题目数量建议不超过：10000题
3. 题目内容长度建议不超过：5000字符
4. 选项内容长度限制：2000字符
5. 导入时检测：超出限制的题目应拒绝导入并返回错误信息
```

4. 答题记录索引
   - INDEX idx_record_id (exam_record_id)  -- 查询某次考试的所有答题记录

5. 选项表索引
   - INDEX idx_question_id (question_id)  -- 查询某题的所有选项

6. 试卷快照索引
   - UNIQUE INDEX uk_record (exam_record_id)  -- 唯一索引，保证一对一

7. 错题本索引
   - INDEX idx_user_question (user_id, question_id)  -- 防止重复收录
   - INDEX idx_user_bank (user_id, bank_id)  -- 按题库查询错题
```

### 13.2 考试状态机

```
                    ┌─────────┐
         开始考试    │ 进行中  │    超时/提交/强制
         ──────────►│  (0)   │◄──────────────────
                    └────┬────┘
                         │
                    提交/超时/强制
                         │
                         ▼
                    ┌─────────┐     查看成绩     ┌─────────┐
                    │ 已提交  │────────────────►│ 已结束  │
                    │   (1)   │                  │   (2)   │
                    └─────────┘                  └─────────┘
```

**状态说明：**
| 状态码 | 名称 | 说明 |
|--------|------|------|
| 0 | 进行中 | 考生正在答题 |
| 1 | 已提交 | 考生已提交（含主动提交、超时自动提交、强制提交），**等待查看成绩** |
| 2 | 已结束 | 考生已查看成绩/解析，考试流程完结 |

**状态流转规则：**
```
1. status=0（进行中）→ status=1（已提交）：
   - 考生主动提交
   - 倒计时超时自动提交
   - 管理员/防作弊系统强制提交

2. status=1（已提交）→ status=2（已结束）：
   - 考生点击"查看成绩"或"查看解析"按钮时触发
   - 系统自动跳转到成绩页面也算作查看

3. status=1（已提交）后：
   - 考生可反复查看成绩和答题详情（不改变状态）
   - 考生点击"查看解析"后才将 status 改为 2（表示流程完结）
```

**提交方式说明（submitType）：**
| 状态码 | 名称 | 说明 |
|--------|------|------|
| 0 | 未提交 | 考试进行中，尚未提交 |
| 1 | 主动提交 | 考生点击交卷按钮提交 |
| 2 | 超时自动提交 | 倒计时结束系统自动提交 |
| 3 | 强制提交 | 管理员强制提交或防作弊系统强制提交 |

### 13.3 超时自动提交定时任务

**扫描策略（避免全表扫描）：**
```
1. 基于 idx_status_end_time 索引，只扫描 status=0 且 end_time < now 的记录
   - 注意：不使用 start_time + duration 做条件（会导致索引失效）
2. 每次扫描批次处理（如每批500条），防止内存溢出
3. 使用分布式锁防止多实例重复扫描：lock:exam:timeout:scan
4. 扫描频率：每10-15秒执行一次（避免考生等待时间过长）
5. 异步处理：将超时记录放入MQ异步评分，降低数据库压力
6. 失败重试：异步任务失败后，使用延迟重试机制（最多3次，间隔 30s/60s/120s）
```

**伪代码：**
```java
@Scheduled(fixedRate = 12000) // 每12秒扫描一次
public void scanTimeoutExams() {
    if (!distributedLock.tryLock("lock:exam:timeout:scan", 60, TimeUnit.SECONDS)) {
        return; // 其他实例正在执行
    }
    try {
        List<ExamRecord> timeoutRecords = examRecordMapper
            .selectTimeoutRecords(500); // LIMIT 500
        for (ExamRecord record : timeoutRecords) {
            // 异步提交
            asyncTaskExecutor.execute(() -> autoSubmit(record));
        }
    } finally {
        distributedLock.unlock("lock:exam:timeout:scan");
    }
}
```

### 13.4 缓存策略

**Redis 缓存设计：**

| 缓存Key | 类型 | TTL | 说明 |
|---------|------|-----|------|
| `exam:{tenantId}:bank:{bankId}` | String | 1小时 | 题库基础信息 |
| `exam:{tenantId}:question:{questionId}` | String | 1小时 | 题目详情（含选项） |
| `exam:{tenantId}:snapshot:{recordId}` | String | 7天 | 试卷快照数据（较大，独立缓存） |
| `exam:{tenantId}:progress:{recordId}` | Hash | 考试时长+30分钟 | 答题进度缓存（field=questionId, value=answer） |
| `exam:{tenantId}:lock:*` | String | 10秒 | 分布式锁 |

**缓存更新策略：**
```
1. 题库/题目修改时 → 删除对应缓存（Cache-Aside 模式）
2. 考试进行中 → 答题进度写入 Redis Hash，每30秒异步落库
3. 考试结束后 → 删除进度缓存，保留快照缓存7天供查看成绩（支持考后多日复盘）
```

**localStorage 键名设计：**
```javascript
// 必须包含 userId，防止同一浏览器多账号登录时数据串台
localStorage.setItem(`exam:progress:${userId}:${recordId}`, JSON.stringify(answers))
```

**localStorage 数据安全（防篡改）：**
```
1. 本地缓存的答案数据需进行简单混淆/签名，防止考生通过 DevTools 直接修改
2. 方案：存储时附加 HMAC-SHA256 签名（密钥由后端生成并缓存在前端内存）
   localStorage.setItem(`exam:progress:${userId}:${recordId}`, 
     JSON.stringify({ answers, sign: hmac(answers, nonce) }))
3. 恢复时校验签名，校验失败则丢弃本地数据，以服务端为准
4. 明确原则：本地缓存仅供恢复体验，最终以服务端保存的答案为准
```

### 13.5 数据归档与容量规划

**答题记录表数据量估算：**
```
假设：100题/卷 × 1万人/场 × 10场/月 = 1000万条/月
一年累计：1.2亿条
按每条 500字节估算：1.2亿 × 500B ≈ 60GB/年
```

**软删关联处理：**
```
1. exam_record 软删（deleted=1）时，级联处理策略：
   - exam_answer：保留（历史答题数据是审计依据，不应随记录删除而物理删除）
   - exam_paper_snapshot：保留（支持查看历史考试详情）
   - exam_behavior_log：保留（防作弊审计需要）
   - 查询接口：exam_record 查询默认过滤 deleted=0，管理端可勾选"显示已删除"
2. exam_question 软删时：
   - 若题目已被历史快照引用：允许软删，快照中的题目内容不受影响
   - 若题目仅在错题本/收藏夹中被引用：软删后展示为"题目已下架"
```

**归档策略：**
```
1. 冷数据定义：status=2（已结束）且 submitTime > 3个月
2. 归档方式：
   ├── 方案A（推荐）：按月分表 exam_answer_202501, exam_answer_202502
   │   └── 查询时根据时间路由到对应表
   └── 方案B：归档到历史库（低性能存储）
3. 保留策略：
   ├── 热库：保留最近3个月数据
   ├── 温库：保留最近1年数据（压缩存储）
   └── 冷库：超过1年的数据可导出为文件存档
```

**其他大表关注：**
| 表名 | 预估增长 | 优化建议 |
|------|----------|----------|
| ExamAnswer | 千万级/月 | 分表或归档 |
| ExamRecord | 万级/月 | 按租户+时间索引 |
| ExamBehaviorLog | 十万级/月 | 保留30天后清理 |
| ExamAuditLog | 万级/月 | 保留90天后清理 |

### 13.6 容灾与降级设计

**故障场景与降级方案：**

| 故障组件 | 影响 | 降级方案 |
|----------|------|----------|
| Redis 宕机 | 分布式锁失效、缓存失效、进度缓存丢失 | 降级为数据库乐观锁（版本号）或悲观锁（SELECT FOR UPDATE）；进度缓存降级为直接写库 |
| MQ 宕机 | 超时自动提交无法异步评分 | 同步处理（降低吞吐量但保证可用）+ 告警通知运维 |

**MQ 在考试系统中的使用场景（明确）：**
```
当前设计中，以下场景建议使用异步消息队列：
1. 超时自动提交后的评分：定时任务扫描到超时记录 → 发送MQ → 消费者异步评分
   - 削峰：避免大量考生同时超时导致数据库评分压力过大
2. 成绩排名计算：考试提交后 → 发送MQ → 异步更新 Redis 排名缓存
3. 错题本收录：批量提交后 → 发送MQ → 异步处理错题收录和统计更新

若未引入MQ，可降级方案：
- 场景1：Spring @Async 线程池异步评分
- 场景2：定时任务每5分钟批量计算排名
- 场景3：提交接口同步处理错题本（事务内完成，简单但耗时增加）
```
| 数据库主从延迟 | 开始考试后立即查询可能读不到记录 | 关键读操作强制走主库（如 `@Transactional` 内查询或强制主库路由） |
| 前端 localStorage 被清除 | 断线续考本地缓存丢失 | 以服务端保存的进度为准（服务端每30秒持久化一次） |
| 快照数据过大 | 网络传输慢、Redis 内存占用高 | 快照数据可压缩存储（gzip）；超过 100KB 建议存文件存储（OSS/MinIO） |

### 13.7 系统监控指标

**核心监控指标：**
```
1. 考试健康度：
   - 进行中考试数量（正常值范围估算）
   - 超时待处理考试数量（>100 需要告警）
   - 提交失败数量（>0 需要告警）

2. 定时任务监控：
   - 超时扫描任务执行情况（每12秒执行）
   - 任务失败次数（连续失败3次发送告警）
   - 分布式锁获取成功率

3. 接口性能：
   - /student-exam/start 平均响应时间（>2秒需要优化）
   - /student-exam/submit 平均响应时间（>5秒需要优化）
   - 错误率（>1% 需要关注）

4. 容量预警：
   - 答题记录表增长率（预估何时达到容量上限）
   - Redis 内存使用率（>70% 需要告警）
```

**告警配置建议：**
| 告警项 | 阈值 | 通知方式 |
|--------|------|----------|
| 超时待处理考试 > 100 | 100 | 钉钉/企微 |
| 超时扫描任务连续失败 | 3次 | 钉钉/企微 |
| 接口错误率 > 1% | 1% | 邮件 |
| Redis 内存 > 70% | 70% | 钉钉/企微 |

### 13.8 灰度发布策略

**新功能上线规则：**
```
1. 数据库变更：始终向后兼容，新增字段提供默认值
2. 接口变更：
   - 新增接口直接全量发布
   - 修改接口先兼容旧逻辑，通过参数控制切换
3. 考试中的试卷不受新功能影响：
   - 正在进行（status=0）的考试使用原有逻辑
   - 新建/修改的试卷可启用新功能
4. 灰度方案：
   - 方案A：按租户灰度（新功能只对部分租户开放）
   - 方案B：按试卷灰度（重要试卷先用旧逻辑）
```

### 13.9 接口规范与限流

**分页参数规范：**
```
pageNo: 当前页码（从1开始，默认1）
pageSize: 每页条数（默认10，最大100，超过100按100处理）
```

**关键接口限流：**
| 接口 | 限流策略 | 说明 |
|------|----------|------|
| /student-exam/start | 同一用户 10秒/1次 | 防止恶意刷考试次数 |
| /student-exam/submit | 同一记录 5秒/1次 | 防止重复提交 |
| /student-exam/save | 同一记录 5秒/1次 | 防止高频自动保存 |
| /question/import | 全系统 1次/分钟 | 防止大数据量导入拖垮系统 |

**提交防重幂等设计：**
```java
// 使用数据库唯一索引实现幂等
UNIQUE INDEX uk_record_submit (exam_record_id, submit_seq)
// 前端提交时生成 submitSeq（UUID），重复提交因唯一索引冲突而失败
```

### 13.10 前端倒计时防篡改

**倒计时实现方案：**
```
1. 前端展示倒计时：基于服务器返回的 endTime 计算剩余秒数
2. 每次保存/提交时，后端校验 serverTime <= endTime
3. 如果前端篡改倒计时延长考试时间 → 后端校验不通过，返回"考试已超时"
4. 前端每30秒同步一次服务器时间，校准本地时钟偏差
5. 时钟偏差告警：如果服务器时间与本地时间差超过 5分钟，提示考生检查本地时间
```

**时间同步机制：**
```
1. 考试开始时：后端返回 serverTime 和 endTime
2. 答题过程中：前端每30秒调用 /student-exam/save，后端返回 serverTime
3. 本地校准：计算出本地时间与服务器时间的偏差 offset = serverTime - localTime
4. 显示倒计时：remainingSeconds = endTime - (localTime + offset)
5. 提交时：后端以 serverTime 为准，不依赖前端传递的时间
```

**接口返回示例：**
```json
{
  "serverTime": "2026-04-20T10:30:00",
  "endTime": "2026-04-20T11:15:00",
  "remainingSeconds": 2700
}
```

### 13.11 关键代码片段

**随机抽题（伪代码）：**
```java
public List<Question> randomQuestions(Long examId) {
    List<Question> result = new ArrayList<>();
    List<ExamBankQuestion> configs = examBankQuestionMapper.selectByExamId(examId);
    for (ExamBankQuestion config : configs) {
        List<Question> questions = questionMapper.selectRandomByBankAndType(
            config.getBankId(),
            config.getQuestionType(),
            config.getQuestionCount()
        );
        result.addAll(questions);
    }
    Collections.shuffle(result); // 随机打乱
    return result;
}
```

**自动评分（伪代码）：**
```java
public int calculateScore(String userAnswer, String correctAnswer, 
                          Integer type, Integer strategy, int fullScore) {
    // 统一处理未作答或空答案
    if (userAnswer == null || userAnswer.trim().isEmpty()) {
        return 0;
    }
    if (type == 1 || type == 3) { // 单选/判断
        return userAnswer.equals(correctAnswer) ? fullScore : 0;
    } else { // 多选
        return calculateMultiScore(
            JSON.parseArray(userAnswer, String.class),
            JSON.parseArray(correctAnswer, String.class),
            strategy, fullScore
        );
    }
}

public int calculateMultiScore(List<String> userAnswer, List<String> correctAnswer,
                               Integer strategy, int fullScore) {
    // 1. 处理未作答或空答案
    if (userAnswer == null || userAnswer.isEmpty()) {
        return 0; // 未作答得零分
    }

    // 2. 检测错选（选了不在正确答案中的选项）
    boolean hasWrongOption = !correctAnswer.containsAll(userAnswer);
    if (hasWrongOption) {
        return 0; // 错选直接零分
    }

    // 3. 正好完全匹配
    if (userAnswer.equals(correctAnswer)) {
        return fullScore;
    }

    // 4. 少选情况
    if (userAnswer.size() < correctAnswer.size()) {
        if (strategy == 1) { // 完全匹配策略：少选零分
            return 0;
        } else if (strategy == 2) { // 少选得半分（整数除法向下取整）
            return fullScore / 2;
        } else if (strategy == 3) { // 按比例得分（整数除法向下取整）
            // ⚠️ 精度警告：5分/3选项=1分/选项，选2个得2分，全选对仅得3分（非5分）
            // 业务上如需精确得分，请将 score 字段改为 DECIMAL(5,1)，此处改用 BigDecimal 计算
            int perScore = fullScore / correctAnswer.size();
            return perScore * userAnswer.size();
        }
    }

    // 5. 多选情况（userAnswer.size() > correctAnswer.size()）
    // 已排除错选，说明多选的都是正确选项
    if (strategy == 1) {
        return 0; // 完全匹配策略：多选零分
    } else if (strategy == 2) {
        return 0; // 少选得半分策略：多选零分
    } else if (strategy == 3) {
        return fullScore; // 按比例策略：已排除错选，得满分
    }

    return 0;
}
```

---

## 十四、数据库建表SQL

```sql
-- =============================================
-- 在线题库与考试系统 - 数据库表结构
-- 模块名：yudao-module-exam
-- 建议使用 MySQL 8.0+
-- =============================================

-- ---------------------------------------------
-- 1. 题库表 (exam_bank)
-- ---------------------------------------------
CREATE TABLE exam_bank (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    name            VARCHAR(100)       NOT NULL                  COMMENT '题库名称',
    description     VARCHAR(500)       DEFAULT NULL               COMMENT '题库描述',
    remark          VARCHAR(500)       DEFAULT NULL               COMMENT '备注',
    status          TINYINT UNSIGNED  NOT NULL  DEFAULT 1        COMMENT '状态（0=禁用 1=启用）',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater         VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    deleted         BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_status (status)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='题库表';

-- ---------------------------------------------
-- 2. 题目表 (exam_question)
-- ---------------------------------------------
CREATE TABLE exam_question (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    bank_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '题库ID',
    type            TINYINT UNSIGNED   NOT NULL                  COMMENT '题目类型（1=单选 2=多选 3=判断）',
    content         TEXT               NOT NULL                  COMMENT '题目内容（支持富文本）',
    content_hash    VARCHAR(64)         DEFAULT NULL               COMMENT '题目内容MD5哈希（用于导入时重复检测）',
    parse           TEXT               DEFAULT NULL               COMMENT '题目解析',
    score           INT UNSIGNED       NOT NULL  DEFAULT 1        COMMENT '默认分值',
    correct_answer  VARCHAR(10)        DEFAULT NULL               COMMENT '正确答案（判断题专用：1=对 0=错；单选/多选答案走选项表）',
    difficulty      TINYINT UNSIGNED   DEFAULT NULL               COMMENT '难度（1=简单 2=中等 3=困难）',
    status          TINYINT UNSIGNED   NOT NULL  DEFAULT 1        COMMENT '状态（0=禁用 1=启用）',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater         VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    deleted         BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_tenant_bank (tenant_id, bank_id),
    INDEX idx_bank_type (bank_id, type),
    INDEX idx_bank_hash (bank_id, content_hash),
    INDEX idx_status (status)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='题目表';

-- ---------------------------------------------
-- 3. 选项表 (exam_question_option)
-- ---------------------------------------------
CREATE TABLE exam_question_option (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    question_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '题目ID',
    option_key      VARCHAR(10)        NOT NULL                  COMMENT '选项标识（A/B/C/D）',
    content         VARCHAR(1000)      NOT NULL                  COMMENT '选项内容',
    is_correct      BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否为正确答案',
    sort            INT UNSIGNED       NOT NULL  DEFAULT 0        COMMENT '排序',
    status          TINYINT UNSIGNED   NOT NULL  DEFAULT 1        COMMENT '状态（0=禁用 1=启用）',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_question_option_key (question_id, option_key),
    INDEX idx_question_id (question_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='选项表';

-- ---------------------------------------------
-- 4. 试卷表 (exam_exam)
-- ---------------------------------------------
CREATE TABLE exam_exam (
    id                  BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id           BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    name                VARCHAR(200)       NOT NULL                  COMMENT '试卷名称',
    description         VARCHAR(500)       DEFAULT NULL               COMMENT '试卷描述',
    exam_mode           TINYINT UNSIGNED   NOT NULL  DEFAULT 2        COMMENT '考试模式（1=练习 2=正式）',
    total_score         INT UNSIGNED       NOT NULL  DEFAULT 100      COMMENT '总分',
    passing_score       INT UNSIGNED       NOT NULL  DEFAULT 60       COMMENT '及格分数',
    duration_minutes    INT UNSIGNED       NOT NULL  DEFAULT 60       COMMENT '考试时长（分钟）',
    total_times         INT UNSIGNED       NOT NULL  DEFAULT 0        COMMENT '最大考试次数（0=不限制）',
    valid_start_time    DATETIME           DEFAULT NULL               COMMENT '有效开始时间',
    valid_end_time      DATETIME           DEFAULT NULL               COMMENT '有效结束时间',
    status              TINYINT UNSIGNED   NOT NULL  DEFAULT 1        COMMENT '状态（0=禁用 1=启用）',
    show_answer_mode    TINYINT UNSIGNED   NOT NULL  DEFAULT 1        COMMENT '答案显示模式（1=交卷后 2=结束后 3=不显示）',
    multi_score_strategy TINYINT UNSIGNED   NOT NULL  DEFAULT 1        COMMENT '多选评分策略（1=完全匹配 2=少选得半分 3=按比例）',
    total_score_auto    BIT                NOT NULL  DEFAULT b'1'    COMMENT '总分是否自动计算（1=自动 0=手动）',
    creator             VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time         DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater             VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time         DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    deleted             BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_status (status)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='试卷表';

-- ---------------------------------------------
-- 5. 试卷-题库关联表 (exam_exam_bank_question)
-- ---------------------------------------------
CREATE TABLE exam_exam_bank_question (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷ID',
    bank_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '题库ID',
    question_type   TINYINT UNSIGNED   NOT NULL                  COMMENT '题型（1=单选 2=多选 3=判断）',
    question_count  INT UNSIGNED       NOT NULL                  COMMENT '抽取题目数量',
    per_score       INT UNSIGNED       NOT NULL  DEFAULT 1        COMMENT '每题分值',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_exam_bank_type (exam_id, bank_id, question_type),
    INDEX idx_exam_id (exam_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='试卷-题库关联表';

-- ---------------------------------------------
-- 6. 试卷-允许角色关联表 (exam_exam_role)
-- ---------------------------------------------
CREATE TABLE exam_exam_role (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷ID',
    role_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '角色ID',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_exam_role (exam_id, role_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='试卷-允许角色关联表';

-- ---------------------------------------------
-- 6.1. 试卷-允许用户关联表 (exam_exam_user)
-- ---------------------------------------------
CREATE TABLE exam_exam_user (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '用户ID',
    expire_time     DATETIME           DEFAULT NULL               COMMENT '授权过期时间（为空表示不过期）',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_exam_user (exam_id, user_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='试卷-允许用户关联表';

-- ---------------------------------------------
-- 7. 试卷快照表 (exam_paper_snapshot)
-- ---------------------------------------------
CREATE TABLE exam_paper_snapshot (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷ID',
    exam_record_id  BIGINT UNSIGNED    NOT NULL                  COMMENT '考试记录ID（唯一关联）',
    snapshot_data   LONGTEXT          NOT NULL                  COMMENT '快照数据（JSON格式，建议用LONGTEXT防止200题试卷数据过大）',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_record_id (exam_record_id),
    INDEX idx_exam_id (exam_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='试卷快照表';

-- ---------------------------------------------
-- 8. 考试记录表 (exam_record)
-- ---------------------------------------------
CREATE TABLE exam_record (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '考生ID',
    exam_mode       TINYINT UNSIGNED   NOT NULL  DEFAULT 2        COMMENT '考试模式（1=练习 2=正式）',
    total_score     INT UNSIGNED       DEFAULT NULL               COMMENT '考试成绩',
    correct_count   INT UNSIGNED       DEFAULT NULL               COMMENT '正确题数',
    total_count     INT UNSIGNED       DEFAULT NULL               COMMENT '总题数',
    status          TINYINT UNSIGNED   NOT NULL  DEFAULT 0        COMMENT '状态（0=进行中 1=已提交 2=已结束）',
    submit_type     TINYINT UNSIGNED   NOT NULL  DEFAULT 0        COMMENT '提交方式（0=未提交 1=主动 2=超时 3=强制）',
    cheating_flag   BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否疑似作弊',
    start_time      DATETIME           NOT NULL                  COMMENT '开始时间',
    end_time        DATETIME           NOT NULL                  COMMENT '预计结束时间',
    submit_time     DATETIME           DEFAULT NULL               COMMENT '提交时间',
    ip              VARCHAR(128)        DEFAULT NULL               COMMENT '答题IP',
    user_agent      VARCHAR(500)        DEFAULT NULL               COMMENT '浏览器信息',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater         VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    deleted         BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_exam_user (exam_id, user_id),
    INDEX idx_user_exam (user_id, exam_id),  -- 快速查询某用户对某试卷的考试次数
    INDEX idx_user_status (user_id, status),
    INDEX idx_status_end_time (status, end_time),
    INDEX idx_exam_mode (exam_mode),  -- 按考试模式筛选（练习/正式）
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='考试记录表';

-- ---------------------------------------------
-- 9. 答题记录表 (exam_answer)
-- ---------------------------------------------
CREATE TABLE exam_answer (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_record_id  BIGINT UNSIGNED    NOT NULL                  COMMENT '考试记录ID',
    snapshot_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '试卷快照ID',
    question_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '题目ID',
    question_type   TINYINT UNSIGNED   NOT NULL                  COMMENT '题目类型',
    user_answer     VARCHAR(500)        DEFAULT NULL               COMMENT '用户答案（JSON格式）',
    correct_answer  VARCHAR(500)        DEFAULT NULL               COMMENT '正确答案（JSON格式）',
    answer_status   TINYINT UNSIGNED   NOT NULL  DEFAULT 0        COMMENT '答题状态（0=未答 1=已答）',
    is_correct      BIT                DEFAULT NULL               COMMENT '是否正确',
    score           INT UNSIGNED       DEFAULT NULL               COMMENT '本题得分',
    answer_duration INT UNSIGNED       DEFAULT NULL               COMMENT '答题时长（秒）',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '答题人',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '答题时间',
    PRIMARY KEY (id),
    INDEX idx_record_id (exam_record_id),
    INDEX idx_snapshot_id (snapshot_id),
    INDEX idx_question_id (question_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='答题记录表（题目内容不冗余，通过snapshot_id关联快照获取）';

-- ---------------------------------------------
-- 10. 考试行为日志表 (exam_behavior_log)
-- ---------------------------------------------
CREATE TABLE exam_behavior_log (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_record_id  BIGINT UNSIGNED    NOT NULL                  COMMENT '考试记录ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '用户ID',
    action_type     TINYINT UNSIGNED   NOT NULL                  COMMENT '行为类型（1=切屏 2=复制粘贴 3=窗口失焦 4=异常提交）',
    action_desc     VARCHAR(500)        DEFAULT NULL               COMMENT '行为描述',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '发生时间',
    deleted         BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_record_id (exam_record_id),
    INDEX idx_user_id (user_id),
    INDEX idx_record_time (exam_record_id, create_time)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='考试行为日志表';

-- ---------------------------------------------
-- 11. 错题本表 (exam_wrong_book)
-- ---------------------------------------------
CREATE TABLE exam_wrong_book (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '用户ID',
    question_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '题目ID',
    bank_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '题库ID',
    wrong_count     INT UNSIGNED       NOT NULL  DEFAULT 1        COMMENT '做错次数',
    last_wrong_time DATETIME           NOT NULL                  COMMENT '最近一次做错时间',
    auto_remove     BIT                NOT NULL  DEFAULT b'1'     COMMENT '练习模式答对后是否自动移除（1=自动 0=手动）',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater         VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_user_question (user_id, question_id),
    INDEX idx_user_bank (user_id, bank_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='错题本表';

-- ---------------------------------------------
-- 12. 收藏夹表 (exam_favorite)
-- ---------------------------------------------
CREATE TABLE exam_favorite (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '用户ID',
    question_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '题目ID',
    bank_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '题库ID',
    note            VARCHAR(500)        DEFAULT NULL               COMMENT '收藏备注',
    creator         VARCHAR(64)        DEFAULT ''                 COMMENT '创建者',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updater         VARCHAR(64)        DEFAULT ''                 COMMENT '更新者',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE INDEX uk_user_question (user_id, question_id),
    INDEX idx_user_bank (user_id, bank_id),
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='收藏夹表';

-- ---------------------------------------------
-- 13. 考试通知表 (exam_notification)
-- ---------------------------------------------
CREATE TABLE exam_notification (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    user_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '接收用户ID',
    exam_id         BIGINT UNSIGNED    DEFAULT NULL               COMMENT '关联试卷ID（可选）',
    exam_record_id  BIGINT UNSIGNED    DEFAULT NULL               COMMENT '关联考试记录ID（可选）',
    title           VARCHAR(200)       NOT NULL                  COMMENT '通知标题',
    content         TEXT                DEFAULT NULL               COMMENT '通知内容',
    type            TINYINT UNSIGNED   NOT NULL                  COMMENT '类型（1=考试提醒 2=成绩通知 3=系统公告）',
    read_status     TINYINT UNSIGNED   NOT NULL  DEFAULT 0        COMMENT '已读状态（0=未读 1=已读）',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_user_id (user_id),
    INDEX idx_user_read (user_id, read_status),  -- 查询用户未读消息
    INDEX idx_create_time (create_time)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='考试通知表';

-- ---------------------------------------------
-- 13.1. 防作弊规则表 (exam_cheating_rule)
-- ---------------------------------------------
CREATE TABLE exam_cheating_rule (
    id                      BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id               BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    exam_id                 BIGINT UNSIGNED    DEFAULT NULL               COMMENT '试卷ID（为空表示默认规则）',
    rule_type               TINYINT UNSIGNED   NOT NULL                  COMMENT '规则类型（1=切屏次数 2=答题时间异常 3=复制粘贴）',
    warn_threshold          INT UNSIGNED       NOT NULL  DEFAULT 3        COMMENT '警告阈值',
    force_submit_threshold  INT UNSIGNED       NOT NULL  DEFAULT 5        COMMENT '强制提交阈值（0表示不强制提交）',
    enabled                 BIT                NOT NULL  DEFAULT b'1'    COMMENT '是否启用',
    deleted                 BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    PRIMARY KEY (id),
    INDEX idx_tenant_exam (tenant_id, exam_id)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='防作弊规则表';

-- ---------------------------------------------
-- 13.2. 异步导入任务表 (exam_import_task)
-- ---------------------------------------------
CREATE TABLE exam_import_task (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    operator_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '操作人ID',
    bank_id         BIGINT UNSIGNED    NOT NULL                  COMMENT '目标题库ID',
    file_name       VARCHAR(200)       NOT NULL                  COMMENT '文件名',
    total_count     INT UNSIGNED       DEFAULT 0                 COMMENT '总题数',
    success_count   INT UNSIGNED       DEFAULT 0                 COMMENT '成功数',
    fail_count      INT UNSIGNED       DEFAULT 0                 COMMENT '失败数',
    status          TINYINT UNSIGNED   NOT NULL  DEFAULT 0        COMMENT '状态（0=处理中 1=成功 2=失败 3=超时）',
    error_detail    TEXT                DEFAULT NULL               COMMENT '错误详情（JSON格式）',
    expire_time     DATETIME           NOT NULL                  COMMENT '过期时间',
    deleted         BIT                NOT NULL  DEFAULT b'0'     COMMENT '是否删除',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    INDEX idx_tenant (tenant_id),
    INDEX idx_status (status),
    INDEX idx_expire_time (expire_time)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='异步导入任务表';

-- ---------------------------------------------
-- 14. 操作审计日志表 (exam_audit_log)
-- ---------------------------------------------
CREATE TABLE exam_audit_log (
    id              BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT  COMMENT '主键ID',
    tenant_id       BIGINT UNSIGNED    NOT NULL                  COMMENT '租户ID',
    operator_id     BIGINT UNSIGNED    NOT NULL                  COMMENT '操作人ID',
    operator_name   VARCHAR(100)       DEFAULT NULL               COMMENT '操作人名称',
    action_type     TINYINT UNSIGNED   NOT NULL                  COMMENT '操作类型（1=创建 2=修改 3=删除 4=发布 5=调整成绩 6=批量导入 7=批量删除 8=批量操作）',
    entity_type     VARCHAR(50)         DEFAULT NULL               COMMENT '实体类型',
    entity_id       BIGINT UNSIGNED    DEFAULT NULL               COMMENT '实体ID',
    entity_name     VARCHAR(200)        DEFAULT NULL               COMMENT '实体名称',
    before_value    TEXT                DEFAULT NULL               COMMENT '修改前的值（JSON）',
    after_value     TEXT                DEFAULT NULL               COMMENT '修改后的值（JSON）',
    ip              VARCHAR(128)        DEFAULT NULL               COMMENT '操作IP',
    user_agent      VARCHAR(500)        DEFAULT NULL               COMMENT '浏览器信息',
    create_time     DATETIME          NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '操作时间',
    PRIMARY KEY (id),
    INDEX idx_tenant_id (tenant_id),
    INDEX idx_operator_id (operator_id),
    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_create_time (create_time)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COLLATE=utf8mb4_unicode_ci  COMMENT='操作审计日志表';
```

### 索引设计说明

| 索引名称 | 表 | 索引字段 | 说明 |
|---------|-----|----------|------|
| idx_tenant_id | exam_bank/exam_question/exam_exam/exam_answer | tenant_id | 租户隔离查询 |
| idx_bank_type | exam_question | bank_id, type | 按题库+题型查询题目 |
| idx_exam_user | exam_record | exam_id, user_id | 查询某用户对某试卷的考试记录 |
| idx_user_exam | exam_record | user_id, exam_id | 快速统计某用户对某试卷的已完成考试次数 |
| idx_exam_mode | exam_record | exam_mode | 按考试模式筛选（练习/正式） |
| idx_status_end_time | exam_record | status, end_time | **核心索引**：超时扫描，查找进行中且已到结束时间的记录 |
| idx_record_id | exam_answer | exam_record_id | 查询某次考试的所有答题记录 |
| uk_user_question | exam_wrong_book/exam_favorite | user_id, question_id | 防止同一用户重复收录同一错题/收藏 |
| uk_record_id | exam_paper_snapshot | exam_record_id | 唯一约束，保证快照与考试记录一对一 |
| uk_question_option_key | exam_question_option | question_id, option_key | 同一题目的选项标识必须唯一 |
| uk_exam_bank_type | exam_exam_bank_question | exam_id, bank_id, question_type | 同一试卷+题库+题型只能一条配置 |
| idx_record_time | exam_behavior_log | exam_record_id, create_time | 按考试记录查询行为日志并排序 |
| idx_user_read | exam_notification | user_id, read_status | 查询用户的未读通知列表 |

### 字段类型选择说明

| 字段类型 | 选择原因 |
|----------|----------|
| BIGINT UNSIGNED | 主键和外键使用无符号大整型，支持更大的数据量 |
| MEDIUMTEXT | 试卷快照可能达到50KB+，普通TEXT只有64KB上限 |
| TEXT | 题目内容和解析使用TEXT，富文本（HTML标签+图片URL）可能超过VARCHAR上限 |
| VARCHAR(1000) | 选项内容支持富文本，1000字符满足常规选项长度 |
| BIT | 布尔字段使用BIT类型，节省存储空间 |
| TINYINT UNSIGNED | 状态码、类型码等小整数使用，范围0-255 |

---

## 十五、设计总结

### 核心功能一览

| 功能模块 | 功能点 |
|----------|--------|
| 题库管理 | 创建/编辑/删除/查询题库，Excel批量导入导出 |
| 题目管理 | 支持单选/多选/判断题，选项管理，富文本/图片，难度标签 |
| 试卷管理 | 配置题型数量、及格分、考试时长、允许角色、多选评分策略、答案显示模式 |
| 考试模式 | 练习模式（无限次、即时看解析） + 正式考试（限次计时、防作弊） |
| 在线答题 | 整卷显示、实时保存、倒计时、断线续考、本地缓存 |
| 自动评分 | 多题型自动批改、多选部分得分、即时出分 |
| 答题记录 | 完整复盘、查看正确答案与解析（基于快照，不受题库修改影响） |
| 角色权限 | 多角色支持、独立权限模式 |
| 成绩统计 | 及格率、平均分、分数段分布、单题正确率、排名 |
| 防作弊 | 切屏检测、单设备限制、选项乱序、行为日志 |
| 错题本 | 自动收录错题、反复练习、收藏夹 |
| 消息通知 | 考试提醒、成绩通知、超时提醒 |

### 实体清单

```
exam_module/
├── ExamBank           # 题库
├── Question           # 题目
├── QuestionOption     # 选项（单选/多选）
├── Exam               # 试卷
├── ExamBankQuestion   # 试卷-题库关联
├── ExamRole           # 试卷-角色关联
├── ExamPaperSnapshot  # 试卷快照（考试历史数据一致性）
├── ExamRecord         # 考试记录
├── ExamAnswer         # 答题记录
├── ExamBehaviorLog    # 考试行为日志（防作弊）
├── WrongBook          # 错题本
├── FavoriteQuestion   # 收藏夹
├── ExamNotification   # 考试通知/消息
├── ExamCheatingRule   # 防作弊规则配置
├── ExamImportTask     # 异步导入任务
└── ExamAuditLog       # 操作审计日志
```
