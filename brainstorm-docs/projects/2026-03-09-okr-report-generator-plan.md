# OKR报告生成器改造实现计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将 period-report-generator skill 从旧版"工作/学习"任务统计改造为新版"日OKR管理"统计，支持Objective、今日待办、临时任务、KR表格的多维度统计分析

**Architecture:** 保持原有数据流架构（解析→分析→报告生成），扩展数据模型以支持OKR结构，新增优先级和月OKR分组统计维度，替换原有正则表达式匹配逻辑

**Tech Stack:** Python 3.12, 正则表达式, Markdown解析, ASCII图表生成

---

## 前置信息

### 新旧格式对比

**旧格式（当前代码支持的）：**
```markdown
### 2. 工作/学习
- [x] 任务1
- [ ] 任务2
```

**新格式（需要支持的）：**
```markdown
### 1. 日OKR管理
#### 🎯 Objective：完成XX功能
**今日核心目标**：实现核心模块

##### 关键结果（Key Results）
| 序号  | 关键任务 | 关联月OKR | 优先级 | 状态  |
| :-: | :--- | :--- | :-: | :-: |
| KR1 | 新建分支 | [[2026-03月OKR\|月OKR]] | P2 | 已完成 |

##### 今日待办事项
- [x] 任务1
- [ ] 任务2

#### 临时任务
- [x] 临时任务1
```

### 状态值映射
- "已完成" → 计入完成
- "进行中" → 不计入完成，单独统计
- "已取消" → 不计入完成，单独统计

### 目标文件
- 主文件: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py` (1380行)

---

## Task 1: 创建OKR数据解析函数

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:398-417`

**Step 1: 添加OKR解析函数（在parse_task_completion函数之前）**

添加新函数 `parse_okr_data()`:

```python
def parse_okr_data(content: str) -> dict:
    """
    解析日OKR管理数据

    Returns:
        {
            'objective_set': bool,
            'objective_content': str,
            'todo_total': int,
            'todo_completed': int,
            'temp_total': int,
            'temp_completed': int,
            'kr_total': int,
            'kr_completed': int,
            'kr_in_progress': int,
            'kr_cancelled': int,
            'kr_by_priority': dict,
            'kr_by_month_okr': dict,
            'total_tasks': int,
            'total_completed': int
        }
    """
    result = {
        'objective_set': False,
        'objective_content': '',
        'todo_total': 0,
        'todo_completed': 0,
        'temp_total': 0,
        'temp_completed': 0,
        'kr_total': 0,
        'kr_completed': 0,
        'kr_in_progress': 0,
        'kr_cancelled': 0,
        'kr_by_priority': {'P0': {'total': 0, 'completed': 0},
                          'P1': {'total': 0, 'completed': 0},
                          'P2': {'total': 0, 'completed': 0}},
        'kr_by_month_okr': {},
        'total_tasks': 0,
        'total_completed': 0
    }

    # 提取日OKR管理部分
    okr_section = re.search(r'### 1\. 日OKR管理\s*\n(.*?)(?=###|\n##)', content, re.DOTALL)
    if not okr_section:
        return result

    okr_content = okr_section.group(1)

    # 提取Objective
    objective_match = re.search(r'####\s*🎯\s*Objective[：:]\s*(\S+)', okr_content)
    if objective_match:
        objective_text = objective_match.group(1).strip()
        if objective_text and objective_text != '::':
            result['objective_set'] = True
            result['objective_content'] = objective_text

    # 提取今日待办事项复选框
    todo_section = re.search(r'#####\s*今日待办事项\s*\n(.*?)(?=####|#####|\n##|$)', okr_content, re.DOTALL)
    if todo_section:
        todo_content = todo_section.group(1)
        todo_tasks = re.findall(r'- \[([ x])\]', todo_content)
        result['todo_total'] = len(todo_tasks)
        result['todo_completed'] = sum(1 for t in todo_tasks if t == 'x')

    # 提取临时任务复选框
    temp_section = re.search(r'####\s*临时任务\s*\n(.*?)(?=####|#####|\n##|$)', okr_content, re.DOTALL)
    if temp_section:
        temp_content = temp_section.group(1)
        temp_tasks = re.findall(r'- \[([ x])\]', temp_content)
        result['temp_total'] = len(temp_tasks)
        result['temp_completed'] = sum(1 for t in temp_tasks if t == 'x')

    # 提取KR表格
    kr_table_match = re.search(
        r'\|\s*序号\s*\|\s*关键任务.*?\|\s*关联月OKR.*?\|\s*优先级.*?\|\s*状态\s*\|\s*\n'
        r'\|[-:\s\|]+\|\s*\n'
        r'((?:\|[^\n]+\|\s*\n?)*)',
        okr_content, re.DOTALL
    )

    if kr_table_match:
        kr_rows = kr_table_match.group(1).strip().split('\n')
        for row in kr_rows:
            row = row.strip()
            if not row or not row.startswith('|'):
                continue

            parts = [p.strip() for p in row.split('|')[1:-1]]
            if len(parts) >= 5:
                kr_id = parts[0]
                kr_task = parts[1]
                month_okr = parts[2]
                priority = parts[3]
                status = parts[4]

                result['kr_total'] += 1

                # 统计状态
                if status == '已完成':
                    result['kr_completed'] += 1
                elif status == '进行中':
                    result['kr_in_progress'] += 1
                elif status == '已取消':
                    result['kr_cancelled'] += 1

                # 按优先级统计
                if priority in result['kr_by_priority']:
                    result['kr_by_priority'][priority]['total'] += 1
                    if status == '已完成':
                        result['kr_by_priority'][priority]['completed'] += 1

                # 按关联月OKR统计
                month_okr_key = month_okr if month_okr else '未关联'
                if month_okr_key not in result['kr_by_month_okr']:
                    result['kr_by_month_okr'][month_okr_key] = {
                        'total': 0, 'completed': 0, 'in_progress': 0, 'cancelled': 0
                    }
                result['kr_by_month_okr'][month_okr_key]['total'] += 1
                if status == '已完成':
                    result['kr_by_month_okr'][month_okr_key]['completed'] += 1
                elif status == '进行中':
                    result['kr_by_month_okr'][month_okr_key]['in_progress'] += 1
                elif status == '已取消':
                    result['kr_by_month_okr'][month_okr_key]['cancelled'] += 1

    # 计算汇总
    result['total_tasks'] = result['todo_total'] + result['temp_total'] + result['kr_total']
    result['total_completed'] = result['todo_completed'] + result['temp_completed'] + result['kr_completed']

    return result
```

**Step 2: 测试新函数**

创建测试用例验证解析逻辑:

```python
# 临时测试代码（添加到文件末尾或单独测试）
test_content = """
### 1. 日OKR管理
#### 🎯 Objective：完成核心功能开发
**今日核心目标**：实现OKR统计模块

##### 关键结果（Key Results）
| 序号  | 关键任务           | 关联月OKR | 优先级 | 状态  |
| :-: | :------------- | :--- | :-: | :-: |
| KR1 | 新建分支确认修改 | [[2026-03月OKR|月OKR]] | P2 | 已完成 |
| KR2 | 编写核心代码     | [[2026-03月OKR|月OKR]] | P1 | 进行中 |
| KR3 | 废弃的需求       | [[2026-Q1|季度]] | P0 | 已取消 |

##### 今日待办事项
- [x] 阅读文档
- [ ] 编写代码
- [x] 测试功能

#### 临时任务
- [ ] 临时任务1
- [x] 临时任务2
"""

result = parse_okr_data(test_content)
print("Objective设定:", result['objective_set'])
print("今日待办:", result['todo_completed'], "/", result['todo_total'])
print("临时任务:", result['temp_completed'], "/", result['temp_total'])
print("KR完成:", result['kr_completed'], "/", result['kr_total'])
print("P1优先级KR:", result['kr_by_priority']['P1'])
```

运行测试验证输出是否符合预期。

**Step 3: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): add OKR data parsing function with priority and month-okr grouping"
```

---

## Task 2: 修改parse_task_completion函数使用新解析逻辑

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:398-417`

**Step 1: 修改parse_task_completion函数**

将原函数修改为调用新的OKR解析函数，并保持向后兼容的数据结构:

```python
def parse_task_completion(content: str) -> dict:
    """
    解析任务完成情况（适配新版日OKR管理格式）

    Returns:
        {
            'planned': int,           # 今日待办总数
            'completed': int,         # 今日待办完成数
            'work_total': int,        # 总任务数（临时+KR）
            'work_completed': int,    # 总完成数（临时+KR完成）
            'okr_data': dict          # 完整的OKR数据
        }
    """
    # 调用新的OKR解析函数
    okr_data = parse_okr_data(content)

    # 构建兼容旧版的数据结构
    result = {
        'planned': okr_data['todo_total'],
        'completed': okr_data['todo_completed'],
        'work_total': okr_data['temp_total'] + okr_data['kr_total'],
        'work_completed': okr_data['temp_completed'] + okr_data['kr_completed'],
        'okr_data': okr_data  # 保留完整OKR数据供后续使用
    }

    return result
```

**Step 2: 验证修改**

确保函数返回的数据结构包含所有必要字段。

**Step 3: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "refactor(okr): update parse_task_completion to use new OKR parser"
```

---

## Task 3: 修改analyze_goal_completion函数支持OKR数据

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:585-609`

**Step 1: 修改analyze_goal_completion函数**

```python
def analyze_goal_completion(data: list) -> dict:
    """
    分析OKR目标完成情况

    Args:
        data: 任务数据列表（包含okr_data字段）

    Returns:
        统计分析结果
    """
    # 汇总所有OKR数据
    total_todo = sum(d['okr_data']['todo_total'] for d in data if 'okr_data' in d)
    total_todo_completed = sum(d['okr_data']['todo_completed'] for d in data if 'okr_data' in d)

    total_temp = sum(d['okr_data']['temp_total'] for d in data if 'okr_data' in d)
    total_temp_completed = sum(d['okr_data']['temp_completed'] for d in data if 'okr_data' in d)

    total_kr = sum(d['okr_data']['kr_total'] for d in data if 'okr_data' in d)
    total_kr_completed = sum(d['okr_data']['kr_completed'] for d in data if 'okr_data' in d)
    total_kr_in_progress = sum(d['okr_data']['kr_in_progress'] for d in data if 'okr_data' in d)
    total_kr_cancelled = sum(d['okr_data']['kr_cancelled'] for d in data if 'okr_data' in d)

    # 汇总优先级统计
    kr_by_priority = {'P0': {'total': 0, 'completed': 0},
                     'P1': {'total': 0, 'completed': 0},
                     'P2': {'total': 0, 'completed': 0}}

    for d in data:
        if 'okr_data' not in d:
            continue
        for p in ['P0', 'P1', 'P2']:
            kr_by_priority[p]['total'] += d['okr_data']['kr_by_priority'][p]['total']
            kr_by_priority[p]['completed'] += d['okr_data']['kr_by_priority'][p]['completed']

    # 汇总月OKR统计
    kr_by_month_okr = {}
    for d in data:
        if 'okr_data' not in d:
            continue
        for okr_name, stats in d['okr_data']['kr_by_month_okr'].items():
            if okr_name not in kr_by_month_okr:
                kr_by_month_okr[okr_name] = {'total': 0, 'completed': 0, 'in_progress': 0, 'cancelled': 0}
            kr_by_month_okr[okr_name]['total'] += stats['total']
            kr_by_month_okr[okr_name]['completed'] += stats['completed']
            kr_by_month_okr[okr_name]['in_progress'] += stats['in_progress']
            kr_by_month_okr[okr_name]['cancelled'] += stats['cancelled']

    # 计算完成率
    total_tasks = total_todo + total_temp + total_kr
    total_completed = total_todo_completed + total_temp_completed + total_kr_completed

    result = {
        # 兼容旧字段
        'planned_rate': total_todo_completed / total_todo if total_todo > 0 else 0,
        'work_rate': (total_temp_completed + total_kr_completed) / (total_temp + total_kr) if (total_temp + total_kr) > 0 else 0,
        'total_planned': total_todo,
        'total_completed': total_todo_completed,
        'total_work': total_temp + total_kr,
        'total_work_completed': total_temp_completed + total_kr_completed,

        # 新的OKR统计字段
        'todo_total': total_todo,
        'todo_completed': total_todo_completed,
        'todo_rate': total_todo_completed / total_todo if total_todo > 0 else 0,

        'temp_total': total_temp,
        'temp_completed': total_temp_completed,
        'temp_rate': total_temp_completed / total_temp if total_temp > 0 else 0,

        'kr_total': total_kr,
        'kr_completed': total_kr_completed,
        'kr_in_progress': total_kr_in_progress,
        'kr_cancelled': total_kr_cancelled,
        'kr_rate': total_kr_completed / total_kr if total_kr > 0 else 0,

        'kr_by_priority': kr_by_priority,
        'kr_by_month_okr': kr_by_month_okr,

        'total_tasks': total_tasks,
        'overall_rate': total_completed / total_tasks if total_tasks > 0 else 0
    }

    return result
```

**Step 2: 验证修改**

**Step 3: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): enhance analyze_goal_completion with OKR breakdown stats"
```

---

## Task 4: 修改build_task_completion_ascii函数生成OKR图表

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:250-295`

**Step 1: 修改build_task_completion_ascii函数**

```python
def build_task_completion_ascii(title: str, data: dict) -> str:
    """构建OKR完成率图ASCII文本"""
    labels = data.get('labels', [])
    todo_rates = data.get('todo_rates', [])
    temp_rates = data.get('temp_rates', [])
    kr_rates = data.get('kr_rates', [])

    if not labels:
        return f"{title}:\n无数据"

    chart_lines = [f"{title}", "=" * 40]

    # 构建任务类型完成率图
    chart_lines.append("")
    chart_lines.append("📋 今日待办完成率")
    for i, label in enumerate(labels):
        rate = todo_rates[i] if i < len(todo_rates) else 0
        bar_len = int(rate * 25)
        bar = "█" * bar_len + "░" * (25 - bar_len)
        chart_lines.append(f"{label:6} │{bar} {rate*100:5.1f}%")

    chart_lines.append("")
    chart_lines.append("⚡ 临时任务完成率")
    for i, label in enumerate(labels):
        rate = temp_rates[i] if i < len(temp_rates) else 0
        bar_len = int(rate * 25)
        bar = "█" * bar_len + "░" * (25 - bar_len)
        chart_lines.append(f"{label:6} │{bar} {rate*100:5.1f}%")

    chart_lines.append("")
    chart_lines.append("🎯 KR关键结果完成率")
    for i, label in enumerate(labels):
        rate = kr_rates[i] if i < len(kr_rates) else 0
        bar_len = int(rate * 25)
        bar = "█" * bar_len + "░" * (25 - bar_len)
        chart_lines.append(f"{label:6} │{bar} {rate*100:5.1f}%")

    chart_lines.append("=" * 40)
    return "\n".join(chart_lines)
```

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update task completion ASCII chart for OKR breakdown"
```

---

## Task 5: 添加按优先级统计的ASCII图表函数

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py`（在build_task_completion_ascii后添加）

**Step 1: 添加新函数build_okr_priority_ascii**

```python
def build_okr_priority_ascii(title: str, data: dict) -> str:
    """构建KR按优先级完成率图ASCII文本"""
    kr_by_priority = data.get('kr_by_priority', {})

    if not kr_by_priority or all(p['total'] == 0 for p in kr_by_priority.values()):
        return f"{title}:\n无KR数据"

    chart_lines = [f"{title}", "=" * 40]

    priority_labels = {'P0': '🔴 P0(最高)', 'P1': '🟡 P1(高)', 'P2': '🟢 P2(中)'}

    for p in ['P0', 'P1', 'P2']:
        if p in kr_by_priority:
            stats = kr_by_priority[p]
            total = stats['total']
            completed = stats['completed']
            rate = completed / total if total > 0 else 0

            bar_len = int(rate * 25)
            bar = "█" * bar_len + "░" * (25 - bar_len)
            chart_lines.append(f"{priority_labels[p]:8} │{bar} {rate*100:5.1f}% ({completed}/{total})")

    chart_lines.append("=" * 40)
    return "\n".join(chart_lines)
```

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): add ASCII chart for KR completion by priority"
```

---

## Task 6: 修改周报报告模板中的目标完成部分

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:1055-1074`

**Step 1: 替换原有报告模板**

找到周报模板中的目标完成部分（约1055-1074行），替换为:

```python
    report += f"""
## 🎯 OKR目标完成情况

### 任务完成概览

| 任务类型 | 总数 | 已完成 | 进行中 | 已取消 | 完成率 |
|:---|:---:|:---:|:---:|:---:|:---:|
| 📋 今日待办 | {goal_analysis['todo_total']} 项 | {goal_analysis['todo_completed']} 项 | - | - | **{goal_analysis['todo_rate']*100:.1f}%** |
| ⚡ 临时任务 | {goal_analysis['temp_total']} 项 | {goal_analysis['temp_completed']} 项 | - | - | **{goal_analysis['temp_rate']*100:.1f}%** |
| 🎯 关键结果(KR) | {goal_analysis['kr_total']} 项 | {goal_analysis['kr_completed']} 项 | {goal_analysis['kr_in_progress']} 项 | {goal_analysis['kr_cancelled']} 项 | **{goal_analysis['kr_rate']*100:.1f}%** |
| **总计** | {goal_analysis['total_tasks']} 项 | {goal_analysis['total_completed']} 项 | {goal_analysis['kr_in_progress']} 项 | {goal_analysis['kr_cancelled']} 项 | **{goal_analysis['overall_rate']*100:.1f}%** |

### KR按优先级统计

| 优先级 | 总数 | 已完成 | 完成率 |
|:---|:---:|:---:|:---:|
| 🔴 P0(最高) | {goal_analysis['kr_by_priority']['P0']['total']} 项 | {goal_analysis['kr_by_priority']['P0']['completed']} 项 | **{goal_analysis['kr_by_priority']['P0']['completed']/goal_analysis['kr_by_priority']['P0']['total']*100:.1f}%** |
| 🟡 P1(高) | {goal_analysis['kr_by_priority']['P1']['total']} 项 | {goal_analysis['kr_by_priority']['P1']['completed']} 项 | **{goal_analysis['kr_by_priority']['P1']['completed']/goal_analysis['kr_by_priority']['P1']['total']*100:.1f}%** |
| 🟢 P2(中) | {goal_analysis['kr_by_priority']['P2']['total']} 项 | {goal_analysis['kr_by_priority']['P2']['completed']} 项 | **{goal_analysis['kr_by_priority']['P2']['completed']/goal_analysis['kr_by_priority']['P2']['total']*100:.1f}%** |
"""

    # 添加按关联月OKR分组统计
    if goal_analysis['kr_by_month_okr']:
        report += "\n### KR按关联月OKR分组统计\n\n"
        for okr_name, stats in goal_analysis['kr_by_month_okr'].items():
            rate = stats['completed'] / stats['total'] * 100 if stats['total'] > 0 else 0
            display_name = okr_name.replace('[[', '').replace(']]', '').replace('|月OKR', '').replace('|季度', '')
            report += f"""
#### 📌 {display_name}

| 指标 | 数值 |
|:---|:---:|
| KR总数 | {stats['total']} 项 |
| 已完成 | {stats['completed']} 项 |
| 进行中 | {stats['in_progress']} 项 |
| 已取消 | {stats['cancelled']} 项 |
| 完成率 | **{rate:.1f}%** |
"""

    report += f"\n{{charts_section}}\n"
```

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update weekly report template with OKR statistics"
```

---

## Task 7: 修改周报中的总结与建议部分

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:1111-1115`

**Step 1: 更新工作方面的总结**

找到原有代码:
```python
    report += "\n**工作方面**:\n"
    if goal_analysis['work_rate'] >= 0.8:
        report += f"- ✅ 本周工作完成率较高（{goal_analysis['work_rate']*100:.1f}%），任务进展顺利\n"
    else:
        report += f"- ⚠️ 工作完成率{goal_analysis['work_rate']*100:.1f}%，需要调整工作节奏\n"
```

替换为:
```python
    report += "\n**OKR方面**:\n"

    # 今日待办总结
    if goal_analysis['todo_rate'] >= 0.8:
        report += f"- ✅ 今日待办完成率 {goal_analysis['todo_rate']*100:.1f}%，日常任务规划良好\n"
    else:
        report += f"- ⚠️ 今日待办完成率 {goal_analysis['todo_rate']*100:.1f}%，建议合理规划每日任务\n"

    # KR关键结果总结
    if goal_analysis['kr_rate'] >= 0.8:
        report += f"- ✅ 关键结果(KR)完成率 {goal_analysis['kr_rate']*100:.1f}%，核心目标推进顺利\n"
    elif goal_analysis['kr_rate'] >= 0.5:
        report += f"- ⚠️ 关键结果(KR)完成率 {goal_analysis['kr_rate']*100:.1f}%，需要加快核心目标进度\n"
    else:
        report += f"- ❌ 关键结果(KR)完成率 {goal_analysis['kr_rate']*100:.1f}%，核心目标推进缓慢，需要重点关注\n"

    # P0优先级KR总结
    p0_total = goal_analysis['kr_by_priority']['P0']['total']
    p0_completed = goal_analysis['kr_by_priority']['P0']['completed']
    if p0_total > 0:
        p0_rate = p0_completed / p0_total
        if p0_rate < 1.0:
            report += f"- 🔴 有 {p0_total - p0_completed} 个P0级KR未完成，建议优先处理高优先级任务\n"
        else:
            report += f"- ✅ 所有P0级KR已完成\n"
```

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update weekly report summary section with OKR insights"
```

---

## Task 8: 修改月报报告模板中的目标完成部分

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:1242-1253`

**Step 1: 替换月报中的目标完成部分**

找到月报模板中的:
```python
    report += f"""## 🎯 目标完成情况

### 今日计划完成率
- **计划总数**: {goal_analysis['total_planned']} 项
- **已完成**: {goal_analysis['total_completed']} 项
- **完成率**: {goal_analysis['planned_rate']*100:.1f}%

### 工作/学习任务完成率
- **任务总数**: {goal_analysis['total_work']} 项
- **已完成**: {goal_analysis['total_work_completed']} 项
- **完成率**: {goal_analysis['work_rate']*100:.1f}%
"""
```

替换为与周报相同的OKR统计格式（参见Task 6）。

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update monthly report template with OKR statistics"
```

---

## Task 9: 修改月报中的目标达成建议部分

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:1297-1304`

**Step 1: 替换建议部分**

找到原有代码:
```python
    report += "\n### 目标达成建议\n"
    if goal_analysis['planned_rate'] < 0.8:
        report += f"- 今日计划完成率 {goal_analysis['planned_rate']*100:.1f}%，建议合理规划每日任务\n"
    if goal_analysis['work_rate'] < 0.8:
        report += f"- 工作学习完成率 {goal_analysis['work_rate']*100:.1f}%，建议调整工作节奏\n"

    if goal_analysis['planned_rate'] >= 0.8 and goal_analysis['work_rate'] >= 0.8:
        report += "- 各项任务完成情况良好，继续保持！\n"
```

替换为:
```python
    report += "\n### OKR达成建议\n"

    # 今日待办建议
    if goal_analysis['todo_rate'] < 0.7:
        report += f"- 今日待办完成率 {goal_analysis['todo_rate']*100:.1f}%，建议合理规划每日任务量\n"

    # KR关键结果建议
    if goal_analysis['kr_rate'] < 0.6:
        report += f"- 关键结果(KR)完成率 {goal_analysis['kr_rate']*100:.1f}%，核心目标推进偏慢，建议:\n"
        report += "  - 将大目标拆分为更小的可执行任务\n"
        report += "  - 优先处理高优先级(P0/P1)的KR\n"
    elif goal_analysis['kr_rate'] < 0.8:
        report += f"- 关键结果(KR)完成率 {goal_analysis['kr_rate']*100:.1f}%，整体进展良好，可进一步提高效率\n"

    # P0优先级建议
    p0_total = goal_analysis['kr_by_priority']['P0']['total']
    p0_completed = goal_analysis['kr_by_priority']['P0']['completed']
    if p0_total > 0 and p0_completed < p0_total:
        report += f"- 有 {p0_total - p0_completed} 个P0级高优先级KR未完成，建议优先处理\n"

    # 总体评价
    if goal_analysis['overall_rate'] >= 0.8:
        report += "- ✅ 各项OKR任务完成情况良好，继续保持！\n"
```

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update monthly report OKR recommendations section"
```

---

## Task 10: 更新图表生成调用

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:883-896`（周报）
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py:1184`（月报）

**Step 1: 更新周报中的图表数据构建**

找到周报中构建任务图表数据的部分，更新为构建OKR多维数据:

```python
    # 构建OKR趋势图数据
    okr_labels = [d.get('date', '')[5:] for d in task_data]  # MM-DD格式
    todo_rates = []
    temp_rates = []
    kr_rates = []

    for d in task_data:
        okr = d.get('okr_data', {})

        # 今日待办完成率
        todo_total = okr.get('todo_total', 0)
        todo_completed = okr.get('todo_completed', 0)
        todo_rates.append(todo_completed / todo_total if todo_total > 0 else 0)

        # 临时任务完成率
        temp_total = okr.get('temp_total', 0)
        temp_completed = okr.get('temp_completed', 0)
        temp_rates.append(temp_completed / temp_total if temp_total > 0 else 0)

        # KR完成率
        kr_total = okr.get('kr_total', 0)
        kr_completed = okr.get('kr_completed', 0)
        kr_rates.append(kr_completed / kr_total if kr_total > 0 else 0)

    task_data_dict = {
        'labels': okr_labels,
        'todo_rates': todo_rates,
        'temp_rates': temp_rates,
        'kr_rates': kr_rates
    }
```

**Step 2: 添加优先级图表生成**

在图表生成部分添加:

```python
    # 生成KR按优先级完成率图表
    priority_chart_path = charts_dir / "okr_priority.svg"
    priority_data = {'kr_by_priority': goal_analysis['kr_by_priority']}
    if generate_chart_svg('okr_priority', 'KR按优先级完成率', priority_data,
                         priority_chart_path, script_path):
        chart_paths['okr_priority'] = priority_chart_path
```

**Step 3: 更新charts_section构建**

```python
    if 'okr_priority' in chart_paths:
        charts_section += f'### KR按优先级完成率\n\n![KR优先级]({Path(chart_paths["okr_priority"]).name})\n\n'
```

**Step 4: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/scripts/period_report.py
git commit -m "feat(okr): update chart generation calls for OKR data"
```

---

## Task 11: 更新SKILL.md文档

**Files:**
- Modify: `C:/Users/lengm/.claude/skills/period-report-generator/SKILL.md`

**Step 1: 更新技能文档**

更新文档以反映新的OKR格式支持:

- 修改功能描述，说明支持日OKR管理格式
- 更新示例输出，展示OKR统计表格
- 添加OKR格式说明

**Step 2: Commit**

```bash
git add C:/Users/lengm/.claude/skills/period-report-generator/SKILL.md
git commit -m "docs(okr): update SKILL.md with OKR format documentation"
```

---

## Task 12: 端到端测试

**Files:**
- Test: 使用实际日记文件测试

**Step 1: 运行周报生成测试**

```bash
cd "C:/Users/lengm/.claude/skills/period-report-generator"
python scripts/period_report.py weekly --base-dir "D:/myfile/myArticle/my-article" --year 2026 --month 3 --week 10 --output-dir "./test_output"
```

**Step 2: 验证输出内容**

检查生成的周报是否包含:
- [ ] OKR目标完成情况章节
- [ ] 任务完成概览表格（今日待办、临时任务、KR）
- [ ] KR按优先级统计表格
- [ ] KR按关联月OKR分组统计
- [ ] OKR完成率趋势图表

**Step 3: Commit**

```bash
git commit -m "test(okr): add end-to-end testing for OKR report generation"
```

---

## 总结

### 修改点汇总

| 任务 | 修改内容 | 影响范围 |
|:---|:---|:---|
| 1 | 新增parse_okr_data函数 | 新增150行 |
| 2 | 修改parse_task_completion | 重写20行 |
| 3 | 修改analyze_goal_completion | 重写60行 |
| 4 | 修改build_task_completion_ascii | 重写40行 |
| 5 | 新增build_okr_priority_ascii | 新增30行 |
| 6-7 | 更新周报模板 | 修改50行 |
| 8-9 | 更新月报模板 | 修改50行 |
| 10 | 更新图表生成 | 修改30行 |
| 11 | 更新文档 | 修改20行 |
| **总计** | | **约450行** |

### 测试检查清单

- [ ] parse_okr_data正确解析Objective
- [ ] 正确统计今日待办复选框
- [ ] 正确统计临时任务复选框
- [ ] 正确解析KR表格（序号、任务、月OKR、优先级、状态）
- [ ] 正确识别"已完成"/"进行中"/"已取消"状态
- [ ] 按优先级P0/P1/P2正确分组统计
- [ ] 按关联月OKR正确分组统计
- [ ] 周报正确显示OKR统计表格
- [ ] 月报正确显示OKR统计表格
- [ ] 图表正确生成

---

**计划完成！**

接下来有两种执行方式可供选择：

**1. 当前会话执行** - 我使用 `superpowers:subagent-driven-development` 逐个任务执行，每完成一个任务进行代码审查

**2. 并行会话执行** - 新开一个会话，使用 `superpowers:executing-plans` 批量执行任务

你希望采用哪种方式？