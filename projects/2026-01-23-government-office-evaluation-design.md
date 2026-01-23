# 政府股室德能勤绩评价系统设计文档

**日期**: 2026-01-23
**项目**: evaluate
**设计者**: Claude Code

---

## 一、项目概述

### 1.1 项目目标
开发一个移动端友好的网页应用，供内部员工对政府股室的"德、能、勤、绩"四个维度进行评价。

### 1.2 核心功能
- 19个股室的评价功能
- 四个评价维度：德、能、勤、绩
- 四个评分等级：优秀、较好、一般、差
- 同一IP每天限评价一次
- 评价完的股室当天不再显示，次日恢复
- 统计结果图表可视化展示

### 1.3 评价股室列表
1. 安监办主任
2. 审计室主任
3. 计财股股长
4. 信访接待室主任
5. 督导室主任
6. 后勤办主任
7. 办公室主任
8. 招生考试信息中心主任
9. 行政审批办公室主任
10. 资助中心主任
11. 体卫艺幼股副股长
12. 人事股股长
13. 监察室主任
14. 现代教育技术管理中心主任
15. 基础教育股股长
16. 项目办主任
17. 党办主任
18. 职成教股股长

---

## 二、技术架构

### 2.1 技术选型

| 层级 | 技术栈 | 说明 |
|------|--------|------|
| 前端框架 | Next.js 14+ | App Router，服务端渲染 |
| UI组件 | shadcn/ui + Tailwind CSS | 移动端友好 |
| 数据库 | PostgreSQL | 关系型数据库 |
| ORM | Prisma | 类型安全的数据库访问 |
| 图表库 | Recharts | React友好的图表库 |
| 部署 | Vercel / 自建服务器 | 灵活部署选项 |

### 2.2 项目结构

```
evaluate/
├── app/                         # Next.js App Router
│   ├── page.tsx                # 首页：股室列表
│   ├── evaluate/
│   │   └── [id]/
│   │       └── page.tsx       # 评价页面
│   ├── results/
│   │   └── page.tsx           # 统计结果页面
│   └── api/                   # API路由
│       ├── offices/
│       ├── evaluations/
│       └── stats/
├── components/                  # React组件
│   ├── office-card.tsx        # 股室卡片
│   ├── evaluation-form.tsx    # 评价表单
│   └── stats-chart.tsx        # 统计图表
├── lib/                        # 工具函数
│   ├── db.ts                  # 数据库客户端
│   └── utils.ts               # 通用工具
├── prisma/                     # Prisma配置
│   ├── schema.prisma          # 数据模型
│   └── seed.ts                # 初始数据
├── public/                     # 静态资源
└── docs/
    └── plans/                 # 设计文档
```

---

## 三、数据库设计

### 3.1 数据表结构

#### offices（股室表）

```sql
CREATE TABLE offices (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    display_order INT NOT NULL DEFAULT 0,
    created_at    TIMESTAMP DEFAULT NOW()
);
```

#### evaluations（评价记录表）

```sql
CREATE TABLE evaluations (
    id          SERIAL PRIMARY KEY,
    office_id   INT REFERENCES offices(id) ON DELETE CASCADE,
    ip_address  VARCHAR(45) NOT NULL,
    virtue      VARCHAR(10) NOT NULL,    -- 德：优秀/较好/一般/差
    ability     VARCHAR(10) NOT NULL,    -- 能：优秀/较好/一般/差
    diligence   VARCHAR(10) NOT NULL,    -- 勤：优秀/较好/一般/差
    performance VARCHAR(10) NOT NULL,    -- 绩：优秀/较好/一般/差
    evaluated_at DATE NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_evaluations_ip_office_date ON evaluations(ip_address, office_id, evaluated_at);
```

### 3.2 Prisma Schema

```prisma
model Office {
    id           Int     @id @default(autoincrement())
    name         String  @db.VarChar(100)
    displayOrder Int     @default(0) @map("display_order")
    evaluations  Evaluation[]
    createdAt    DateTime @default(now()) @map("created_at")

    @@map("offices")
}

model Evaluation {
    id          Int      @id @default(autoincrement())
    officeId    Int      @map("office_id")
    office      Office   @relation(fields: [officeId], references: [id], onDelete: Cascade)
    ipAddress   String   @map("ip_address") @db.VarChar(45)
    virtue      String   @db.VarChar(10)
    ability     String   @db.VarChar(10)
    diligence   String   @db.VarChar(10)
    performance String   @db.VarChar(10)
    evaluatedAt DateTime @map("evaluated_at") @db.Date
    createdAt   DateTime @default(now()) @map("created_at")

    @@unique([officeId, ipAddress, evaluatedAt], name: "unique_evaluation_per_day")
    @@index([ipAddress, officeId, evaluatedAt], name: "idx_evaluations_ip_office_date")
    @@map("evaluations")
}
```

### 3.3 初始数据

```typescript
// prisma/seed.ts
const offices = [
    "安监办主任", "审计室主任", "计财股股长", "信访接待室主任",
    "督导室主任", "后勤办主任", "办公室主任", "招生考试信息中心主任",
    "行政审批办公室主任", "资助中心主任", "体卫艺幼股副股长", "人事股股长",
    "监察室主任", "现代教育技术管理中心主任", "基础教育股股长",
    "项目办主任", "党办主任", "职成教股股长"
];
```

---

## 四、API设计

### 4.1 API路由清单

| 方法 | 路由 | 功能 |
|------|------|------|
| GET | /api/offices | 获取可评价的股室列表 |
| GET | /api/offices/[id] | 获取单个股室信息 |
| POST | /api/evaluations | 提交评价 |
| GET | /api/stats/[officeId] | 获取指定股室统计 |
| GET | /api/stats | 获取所有股室概览统计 |

### 4.2 API详细设计

#### GET /api/offices
获取当前IP今日可评价的股室列表。

**响应示例**:
```json
{
    "offices": [
        { "id": 1, "name": "安监办主任", "displayOrder": 1 },
        { "id": 2, "name": "审计室主任", "displayOrder": 2 }
    ]
}
```

#### POST /api/evaluations
提交评价记录。

**请求体**:
```json
{
    "office_id": 1,
    "virtue": "优秀",
    "ability": "较好",
    "diligence": "一般",
    "performance": "优秀"
}
```

**响应**:
```json
{
    "success": true,
    "message": "评价提交成功"
}
```

#### GET /api/stats/[officeId]
获取指定股室的详细统计。

**响应示例**:
```json
{
    "officeName": "安监办主任",
    "totalPeople": 15,
    "totalCount": 18,
    "virtue": { "优秀": 10, "较好": 5, "一般": 2, "差": 1 },
    "ability": { "优秀": 8, "较好": 6, "一般": 3, "差": 1 },
    "diligence": { "优秀": 12, "较好": 4, "一般": 1, "差": 1 },
    "performance": { "优秀": 9, "较好": 5, "一般": 3, "差": 1 }
}
```

---

## 五、页面与交互设计

### 5.1 页面结构

#### 首页 (app/page.tsx)
- 标题：政府股室德能勤绩评价系统
- 股室列表（卡片式布局）
- "查看统计结果"链接

#### 评价页面 (app/evaluate/[id]/page.tsx)
- 股室名称
- 四个评价维度表单
- 提交/返回按钮

#### 统计页面 (app/results/page.tsx)
- 股室选择器
- 评价人数/人次统计
- 四个维度的柱状图

### 5.2 交互流程

```
1. 用户打开首页
   ↓
2. 查看今日可评价的股室列表
   ↓
3. 点击股室卡片 → 进入评价页面
   ↓
4. 选择四个维度的评分 → 提交
   ↓
5. 返回首页（已评价股室不再显示）
   ↓
6. 可随时查看统计结果
```

### 5.3 显示逻辑

- **首次访问**: 显示全部19个股室
- **评价后返回**: 已评价的股室不显示
- **全部评价完**: 显示"感谢参与"提示
- **第二天**: 所有股室重新显示

---

## 六、错误处理与边界情况

### 6.1 错误场景处理

| 场景 | 处理方式 |
|------|----------|
| 今日已评价 | 返回提示"您今天已评价过该股室" |
| 股室不存在 | 返回404错误 |
| 提交数据不完整 | 前端验证拦截 |
| IP获取失败 | 记录日志，允许继续（本地测试） |
| 数据库连接失败 | 返回500，显示友好错误页 |

### 6.2 边界情况

- **所有股室评价完成**: 显示完成提示
- **跨天重置**: 使用服务器时间判断日期
- **统计无数据**: 显示"暂无评价数据"
- **并发提交**: 数据库事务保证一致性

---

## 七、部署与环境配置

### 7.1 环境变量

```env
DATABASE_URL="postgresql://user:password@localhost:5432/evaluate"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

### 7.2 开发环境启动

```bash
# 安装依赖
npm install next@latest
npm install prisma @prisma/client
npm install recharts
npm install -D tailwindcss

# 初始化数据库
npx prisma migrate dev --name init
npx prisma db seed

# 启动开发服务器
npm run dev
```

### 7.3 生产部署

**Vercel部署（推荐）**:
1. 连接GitHub仓库
2. 配置环境变量
3. 使用外部PostgreSQL（Supabase/Neon）
4. 自动部署

**Docker部署**:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npx prisma generate
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

---

## 八、测试策略

### 8.1 功能测试清单

- [ ] 提交完整评价，验证数据保存
- [ ] 提交不完整评价，验证前端拦截
- [ ] 重复评价同一股室，验证错误提示
- [ ] 跨天后重新评价，验证限制重置
- [ ] 首页显示逻辑正确
- [ ] 统计数据计算正确

### 8.2 兼容性测试

**移动端浏览器**:
- iOS Safari
- Android Chrome
- 微信内置浏览器

**屏幕适配**:
- 320px（小屏）
- 375px（常见手机）
- 414px（大屏）

### 8.3 手动测试流程

1. 新用户打开首页，验证显示19个股室
2. 点击进入评价页面，选择评分提交
3. 返回首页，验证已评价股室不显示
4. 继续评价直到全部完成
5. 查看统计结果，验证图表显示正确
6. 第二天访问，验证所有股室重新显示

---

## 九、后续优化方向

1. **用户系统扩展**: 可添加登录功能，支持更精细的权限控制
2. **数据导出**: 支持导出评价数据为Excel/PDF
3. **评价留言**: 允许用户添加文字评价
4. **趋势分析**: 增加时间维度的趋势图表
5. **消息推送**: 评价提醒和结果通知

---

## 十、测试定义与验收标准

### 10.1 核心测试文件结构

```
__tests__/
├── unit/                          # 单元测试
│   ├── offices.test.ts            # 股室相关逻辑测试
│   ├── evaluation.test.ts         # 评价逻辑测试
│   └── stats.test.ts              # 统计计算测试
├── integration/                   # 集成测试
│   ├── api.test.ts                # API接口测试
│   └── pages.test.ts              # 页面交互测试
└── e2e/                           # 端到端测试
    └── evaluation-flow.test.ts    # 完整评价流程
```

### 10.2 单元测试定义

#### 10.2.1 股室数据模型测试

```typescript
// __tests__/unit/offices.test.ts

describe('Office Model', () => {
  test('should create office with valid data', () => {
    const office = new Office({
      id: 1,
      name: '安监办主任',
      displayOrder: 1
    });
    expect(office.id).toBe(1);
    expect(office.name).toBe('安监办主任');
  });

  test('should validate office name is not empty', () => {
    expect(() => new Office({ name: '' })).toThrow('股室名称不能为空');
  });

  test('should validate office name length <= 100', () => {
    const longName = 'a'.repeat(101);
    expect(() => new Office({ name: longName })).toThrow('股室名称不能超过100字符');
  });
});
```

**验收标准**:
- [ ] Office模型创建时验证名称必填
- [ ] Office模型创建时验证名称长度限制
- [ ] displayOrder默认为0
- [ ] 18个股室数据正确初始化

#### 10.2.2 评价数据模型测试

```typescript
// __tests__/unit/evaluation.test.ts

describe('Evaluation Model', () => {
  const validEvaluationData = {
    officeId: 1,
    ipAddress: '192.168.1.1',
    virtue: '优秀',
    ability: '较好',
    diligence: '一般',
    performance: '优秀',
    evaluatedAt: new Date()
  };

  test('should create evaluation with valid data', () => {
    const evaluation = new Evaluation(validEvaluationData);
    expect(evaluation.virtue).toBe('优秀');
    expect(evaluation.ability).toBe('较好');
  });

  test('should validate virtue is one of 4 options', () => {
    expect(() => new Evaluation({ ...validEvaluationData, virtue: '无效选项' }))
      .toThrow('评价选项无效');
  });

  test('should validate all 4 dimensions are required', () => {
    expect(() => new Evaluation({ ...validEvaluationData, virtue: undefined }))
      .toThrow('所有评价维度均为必填');
  });
});
```

**验收标准**:
- [ ] 评价四个维度（德/能/勤/绩）验证必填
- [ ] 每个维度只允许：优秀/较好/一般/差
- [ ] 记录IP地址用于限重判断
- [ ] 记录评价时间用于日期判断

#### 10.2.3 统计计算测试

```typescript
// __tests__/unit/stats.test.ts

describe('Statistics Calculation', () => {
  const mockEvaluations = [
    { virtue: '优秀', ability: '较好', diligence: '优秀', performance: '较好' },
    { virtue: '优秀', ability: '优秀', diligence: '较好', performance: '优秀' },
    { virtue: '较好', ability: '较好', diligence: '一般', performance: '一般' }
  ];

  test('should calculate virtue distribution correctly', () => {
    const result = calculateDistribution(mockEvaluations, 'virtue');
    expect(result.优秀).toBe(2);
    expect(result.较好).toBe(1);
    expect(result.一般).toBe(0);
    expect(result.差).toBe(0);
  });

  test('should return zero counts for empty evaluations', () => {
    const result = calculateDistribution([], 'virtue');
    expect(result.优秀).toBe(0);
    expect(result.较好).toBe(0);
    expect(result.一般).toBe(0);
    expect(result.差).toBe(0);
  });

  test('should calculate total count correctly', () => {
    const result = calculateTotalStats(mockEvaluations);
    expect(result.totalEvaluations).toBe(3);
    expect(result.totalPeople).toBe(3);
  });
});
```

**验收标准**:
- [ ] 统计每个维度的优秀/较好/一般/差分布
- [ ] 统计总评价人次
- [ ] 统计评价人数（按IP去重）
- [ ] 空数据时返回全0

### 10.3 集成测试定义

#### 10.3.1 API接口测试

```typescript
// __tests__/integration/api.test.ts

describe('API Routes', () => {
  describe('GET /api/offices', () => {
    test('should return available offices for today', async () => {
      const response = await request(app)
        .get('/api/offices')
        .set('X-Forwarded-For', '192.168.1.1');

      expect(response.status).toBe(200);
      expect(response.body.offices).toBeDefined();
      expect(Array.isArray(response.body.offices)).toBe(true);
    });

    test('should exclude already evaluated offices', async () => {
      // 预先插入一条今日评价记录
      await prisma.evaluation.create({
        data: {
          officeId: 1,
          ipAddress: '192.168.1.1',
          evaluatedAt: new Date(),
          virtue: '优秀',
          ability: '较好',
          diligence: '一般',
          performance: '优秀'
        }
      });

      const response = await request(app)
        .get('/api/offices')
        .set('X-Forwarded-For', '192.168.1.1');

      const officeIds = response.body.offices.map((o: any) => o.id);
      expect(officeIds).not.toContain(1);
    });
  });

  describe('POST /api/evaluations', () => {
    test('should create evaluation successfully', async () => {
      const response = await request(app)
        .post('/api/evaluations')
        .send({
          office_id: 1,
          virtue: '优秀',
          ability: '较好',
          diligence: '一般',
          performance: '优秀'
        })
        .set('X-Forwarded-For', '192.168.1.100');

      expect(response.status).toBe(200);
      expect(response.body.success).toBe(true);
    });

    test('should reject duplicate evaluation on same day', async () => {
      // 先创建一条记录
      await prisma.evaluation.create({
        data: {
          officeId: 2,
          ipAddress: '192.168.1.100',
          evaluatedAt: new Date(),
          virtue: '优秀',
          ability: '较好',
          diligence: '一般',
          performance: '优秀'
        }
      });

      const response = await request(app)
        .post('/api/evaluations')
        .send({
          office_id: 2,
          virtue: '较好',
          ability: '优秀',
          diligence: '较好',
          performance: '较好'
        })
        .set('X-Forwarded-For', '192.168.1.100');

      expect(response.status).toBe(400);
      expect(response.body.success).toBe(false);
      expect(response.body.message).toContain('今天已评价过');
    });

    test('should validate all dimensions are provided', async () => {
      const response = await request(app)
        .post('/api/evaluations')
        .send({
          office_id: 1,
          virtue: '优秀'
          // 缺少其他维度
        });

      expect(response.status).toBe(400);
    });
  });

  describe('GET /api/stats/:officeId', () => {
    test('should return office statistics', async () => {
      const response = await request(app)
        .get('/api/stats/1');

      expect(response.status).toBe(200);
      expect(response.body.officeName).toBeDefined();
      expect(response.body.virtue).toBeDefined();
      expect(response.body.ability).toBeDefined();
      expect(response.body.diligence).toBeDefined();
      expect(response.body.performance).toBeDefined();
    });

    test('should return 404 for non-existent office', async () => {
      const response = await request(app)
        .get('/api/stats/9999');

      expect(response.status).toBe(404);
    });
  });
});
```

**验收标准**:
- [ ] GET /api/offices 返回今日可评价的股室列表
- [ ] GET /api/offices 排除已评价的股室
- [ ] POST /api/evaluations 成功创建评价
- [ ] POST /api/evaluations 拒绝同日重复评价
- [ ] POST /api/evaluations 验证所有维度必填
- [ ] GET /api/stats/:officeId 返回统计分布
- [ ] GET /api/stats/:officeId 对不存在股室返回404

#### 10.3.2 页面交互测试

```typescript
// __tests__/integration/pages.test.ts

describe('Pages', () => {
  describe('Home Page', () => {
    test('should display 18 office cards', async () => {
      const response = await request(app).get('/');
      expect(response.text).toContain('安监办主任');
      expect(response.text).toContain('审计室主任');
      // ... 验证所有股室名称
    });

    test('should have link to results page', async () => {
      const response = await request(app).get('/');
      expect(response.text).toContain('查看统计结果');
    });
  });

  describe('Evaluation Page', () => {
    test('should display 4 rating dimensions', async () => {
      const response = await request(app).get('/evaluate/1');
      expect(response.text).toContain('德');
      expect(response.text).toContain('能');
      expect(response.text).toContain('勤');
      expect(response.text).toContain('绩');
    });

    test('should have 4 rating options per dimension', async () => {
      const response = await request(app).get('/evaluate/1');
      expect(response.text).toContain('优秀');
      expect(response.text).toContain('较好');
      expect(response.text).toContain('一般');
      expect(response.text).toContain('差');
    });
  });

  describe('Results Page', () => {
    test('should have office selector', async () => {
      const response = await request(app).get('/results');
      expect(response.text).toContain('选择股室');
    });

    test('should display charts', async () => {
      const response = await request(app).get('/results');
      expect(response.text).toContain('图表');
    });
  });
});
```

**验收标准**:
- [ ] 首页显示18个股室名称
- [ ] 首页有"查看统计结果"入口
- [ ] 评价页显示四个维度（德/能/勤/绩）
- [ ] 每个维度有4个评分选项
- [ ] 统计页有股室选择器
- [ ] 统计页显示图表

### 10.4 端到端测试定义

```typescript
// __tests__/e2e/evaluation-flow.test.ts

describe('Evaluation Flow E2E', () => {
  beforeEach(async () => {
    // 清理测试数据
    await prisma.evaluation.deleteMany({
      where: { ipAddress: { startsWith: 'test-ip-' } }
    });
  });

  test('complete evaluation flow', async () => {
    const testIp = 'test-ip-e2e-1';

    // 1. 打开首页
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // 2. 验证显示所有股室
    const officeCards = await page.locator('[class*="office-card"]').count();
    expect(officeCards).toBe(18);

    // 3. 点击第一个股室进入评价页
    await page.locator('[class*="office-card"]').first().click();
    await page.waitForURL(/\/evaluate\/\d+/);

    // 4. 选择评分
    await page.locator('text=优秀').first().click();

    // 5. 提交评价
    await page.locator('button:has-text("提交")').click();

    // 6. 验证返回首页且股室不再显示
    await page.waitForURL('/');
    const remainingCards = await page.locator('[class*="office-card"]').count();
    expect(remainingCards).toBe(17);
  });

  test('daily limit enforcement', async () => {
    const testIp = 'test-ip-e2e-2';

    // 第一次评价成功
    await submitEvaluation(1, testIp);

    // 尝试再次评价同一股室
    await page.goto('/evaluate/1');
    const submitButton = page.locator('button:has-text("提交")');
    await submitButton.click();

    // 验证显示错误提示
    await expect(page.locator('text=今天已评价过')).toBeVisible();
  });

  test('results page shows data', async () => {
    // 先创建一些测试数据
    await createTestEvaluations();

    // 打开统计页
    await page.goto('/results');

    // 选择股室
    await page.locator('select').selectOption('安监办主任');

    // 验证图表显示
    await expect(page.locator('[class*="chart"]').first()).toBeVisible();
  });
});
```

**验收标准**:
- [ ] 完整评价流程：首页→评价页→提交→返回首页
- [ ] 评价后股室从列表消失
- [ ] 同日重复评价被阻止
- [ ] 统计页正确显示图表数据
- [ ] 第二天股室恢复显示

### 10.5 验收标准清单

#### 功能验收标准

| 功能点 | 验收标准 | 优先级 |
|--------|----------|--------|
| 股室列表显示 | 首页显示全部18个股室卡片 | P0 |
| 进入评价页 | 点击股室跳转到评价页面 | P0 |
| 评分表单 | 四个维度各4个评分选项 | P0 |
| 提交评价 | 提交成功保存到数据库 | P0 |
| 限重机制 | 同IP同日同股室只能评价一次 | P0 |
| 隐藏已评价 | 评价后返回首页股室消失 | P0 |
| 统计图表 | 显示各维度评价分布柱状图 | P1 |
| 跨天重置 | 次日所有股室恢复显示 | P1 |

#### 质量验收标准

| 标准 | 验收条件 |
|------|----------|
| 响应式布局 | 手机/平板/桌面均正常显示 |
| 加载速度 | 首页加载 < 2秒 |
| 错误处理 | 网络错误显示友好提示 |
| 数据一致性 | 刷新页面数据保持一致 |

#### 测试覆盖率要求

| 类型 | 覆盖率要求 |
|------|------------|
| 单元测试 | > 80% |
| 集成测试 | 核心API 100% |
| E2E测试 | 关键用户流程 100% |

### 10.6 TDD执行顺序

按照TDD红-绿-重构循环，建议按以下顺序实现：

1. **第一轮**: 数据库层
   - [ ] 定义Prisma Schema
   - [ ] 测试数据库CRUD操作
   - [ ] 验证唯一索引约束

2. **第二轮**: API层
   - [ ] GET /api/offices 接口
   - [ ] POST /api/evaluations 接口
   - [ ] GET /api/stats/:officeId 接口

3. **第三轮**: 页面层
   - [ ] 首页渲染
   - [ ] 评价表单
   - [ ] 统计页面

4. **第四轮**: 限重逻辑
   - [ ] IP获取与记录
   - [ ] 同日验证
   - [ ] 跨天重置

5. **第五轮**: 可视化
   - [ ] 统计图表
   - [ ] 数据聚合计算
