# 沉浸式评价交互系统升级 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将现有简洁的评价系统升级为沉浸式卡片流交互体验，融合游戏化进度展示，提升用户参与度和操作效率。

**Architecture:** 基于 Next.js 14 App Router + Tailwind CSS，采用客户端组件实现流畅动画和手势交互。保持现有 API 结构不变，仅升级前端交互层。

**Tech Stack:** Next.js 14, React 18, Tailwind CSS, Framer Motion (新增), TypeScript

---

## Prerequisites (前置准备)

**Step 0: Install Framer Motion**

```bash
cd D:\develop\project\java\work\evaluate
npm install framer-motion
```

Expected: `framer-motion@^11.x` added to `package.json`

**Step 1: Commit installation**

```bash
git add package.json package-lock.json
git commit -m "chore: install framer-motion for animations"
```

---

## Phase 1: 核心动画库配置

### Task 1: 创建动画配置文件

**Files:**
- Create: `lib/animations.ts`

**Step 1: Create animation constants**

```typescript
// lib/animations.ts
import { Variants } from "framer-motion"

// 卡片进入动画
export const cardVariants: Variants = {
  hidden: { opacity: 0, y: 20, scale: 0.95 },
  visible: (i: number) => ({
    opacity: 1,
    y: 0,
    scale: 1,
    transition: {
      delay: i * 0.1,
      duration: 0.4,
      ease: [0.25, 0.1, 0.25, 1]
    }
  }),
  exit: { opacity: 0, scale: 0.95, transition: { duration: 0.2 } }
}

// 进度条动画
export const progressVariants: Variants = {
  hidden: { width: 0 },
  visible: (width: number) => ({
    width: `${width}%`,
    transition: { duration: 0.8, ease: "easeOut" }
  })
}

// 庆祝纸屑动画
export const confettiVariants: Variants = {
  hidden: { y: -20, opacity: 0 },
  visible: {
    y: [0, -30, 0],
    opacity: [0, 1, 0],
    rotate: [0, 360],
    transition: { duration: 1.5, repeat: Infinity, ease: "easeInOut" }
  }
}

// 页面切换动画
export const pageTransition: Variants = {
  initial: { opacity: 0, x: 20 },
  enter: { opacity: 1, x: 0, transition: { duration: 0.3 } },
  exit: { opacity: 0, x: -20, transition: { duration: 0.2 } }
}
```

**Step 2: Commit**

```bash
git add lib/animations.ts
git commit -m "feat: add animation configuration variants"
```

---

### Task 2: 创建进度显示组件

**Files:**
- Create: `components/progress-bar.tsx`

**Step 1: Write the component**

```typescript
// components/progress-bar.tsx
"use client"

import { motion } from "framer-motion"
import { cn } from "@/lib/utils"
import { progressVariants } from "@/lib/animations"

interface ProgressBarProps {
  current: number
  total: number
  className?: string
  showLabel?: boolean
}

export function ProgressBar({
  current,
  total,
  className,
  showLabel = true
}: ProgressBarProps) {
  const percentage = Math.round((current / total) * 100)

  return (
    <div className={cn("w-full", className)}>
      {showLabel && (
        <div className="flex justify-between items-center mb-2 text-sm">
          <span className="text-gray-600">今日评价进度</span>
          <span className="font-medium text-gray-900">{current}/{total}</span>
        </div>
      )}
      <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
        <motion.div
          className="h-full bg-gradient-to-r from-blue-500 to-blue-600 rounded-full"
          initial="hidden"
          animate="visible"
          custom={percentage}
          variants={progressVariants}
        />
      </div>
      {showLabel && (
        <motion.p
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          className="text-xs text-gray-500 mt-1"
        >
          {percentage}% 完成
        </motion.p>
      )}
    </div>
  )
}
```

**Step 2: Commit**

```bash
git add components/progress-bar.tsx
git commit -m "feat: add progress bar component with animation"
```

---

### Task 3: 创建状态图标组件

**Files:**
- Create: `components/status-icon.tsx`

**Step 1: Write the component**

```typescript
// components/status-icon.tsx
"use client"

import { motion } from "framer-motion"
import { CheckCircle2, Clock, Lock, Trophy } from "lucide-react"
import { cn } from "@/lib/utils"

type Status = "completed" | "pending" | "locked" | "current"

interface StatusIconProps {
  status: Status
  className?: string
}

const statusConfig = {
  completed: {
    icon: CheckCircle2,
    color: "text-green-500",
    bgColor: "bg-green-50",
    label: "已完成"
  },
  pending: {
    icon: Clock,
    color: "text-amber-500",
    bgColor: "bg-amber-50",
    label: "待评价"
  },
  locked: {
    icon: Lock,
    color: "text-gray-400",
    bgColor: "bg-gray-50",
    label: "已锁定"
  },
  current: {
    icon: Clock,
    color: "text-blue-500",
    bgColor: "bg-blue-50",
    label: "进行中"
  }
}

export function StatusIcon({ status, className }: StatusIconProps) {
  const config = statusConfig[status]
  const Icon = config.icon

  return (
    <motion.div
      initial={{ scale: 0 }}
      animate={{ scale: 1 }}
      transition={{ type: "spring", stiffness: 200, damping: 10 }}
      className={cn(
        "inline-flex items-center gap-1.5 px-3 py-1 rounded-full text-xs font-medium",
        config.bgColor,
        config.color,
        className
      )}
    >
      <Icon className="w-4 h-4" />
      <span>{config.label}</span>
    </motion.div>
  )
}

// 完成庆祝图标
export function CompletionIcon({ className }: { className?: string }) {
  return (
    <motion.div
      initial={{ scale: 0, rotate: -180 }}
      animate={{ scale: 1, rotate: 0 }}
      transition={{
        type: "spring",
        stiffness: 200,
        damping: 10,
        delay: 0.2
      }}
      className={cn("inline-flex", className)}
    >
      <Trophy className="w-12 h-12 text-yellow-500" />
    </motion.div>
  )
}
```

**Step 2: Commit**

```bash
git add components/status-icon.tsx
git commit -m "feat: add status icon and completion icon components"
```

---

## Phase 2: 首页沉浸式卡片流

### Task 4: 重构 OfficeCard 组件为沉浸式卡片

**Files:**
- Modify: `components/office-card.tsx`

**Step 1: Replace with enhanced version**

```typescript
// components/office-card.tsx
"use client"

import Link from "next/link"
import { motion } from "framer-motion"
import { cn } from "@/lib/utils"
import { cardVariants } from "@/lib/animations"

interface OfficeCardProps {
  id: number
  name: string
  index: number
  status?: "completed" | "pending" | "current"
  totalOffices?: number
  completedCount?: number
}

export function OfficeCard({
  id,
  name,
  index,
  status = "pending",
  totalOffices = 0,
  completedCount = 0
}: OfficeCardProps) {
  const isCompleted = status === "completed"
  const isCurrent = status === "current"

  return (
    <motion.div
      variants={cardVariants}
      initial="hidden"
      animate="visible"
      exit="exit"
      custom={index}
      layout
    >
      <Link href={`/evaluate/${id}`}>
        <motion.div
          whileHover={{ scale: 1.02, y: -4 }}
          whileTap={{ scale: 0.98 }}
          className={cn(
            "group relative rounded-2xl border-2 p-8 shadow-sm transition-all cursor-pointer",
            isCompleted
              ? "border-green-200 bg-gradient-to-br from-green-50 to-white"
              : isCurrent
              ? "border-blue-300 bg-gradient-to-br from-blue-50 to-white ring-4 ring-blue-100"
              : "border-gray-200 bg-white hover:border-blue-200 hover:shadow-lg"
          )}
        >
          {/* 背景装饰 */}
          <motion.div
            className="absolute inset-0 rounded-2xl opacity-0 group-hover:opacity-100 transition-opacity"
            style={{
              background: "radial-gradient(circle at center, rgba(59, 130, 246, 0.05) 0%, transparent 70%)"
            }}
          />

          {/* 完成标记 */}
          {isCompleted && (
            <motion.div
              initial={{ scale: 0 }}
              animate={{ scale: 1 }}
              className="absolute -top-3 -right-3 w-8 h-8 bg-green-500 rounded-full flex items-center justify-center text-white"
            >
              <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={3} d="M5 13l4 4L19 7" />
              </svg>
            </motion.div>
          )}

          {/* 内容 */}
          <div className="relative">
            {/* 图标区域 */}
            <motion.div
              className="w-16 h-16 mx-auto mb-4 rounded-2xl flex items-center justify-center text-3xl"
              animate={isCurrent ? { scale: [1, 1.05, 1] } : {}}
              transition={{ duration: 2, repeat: Infinity }}
              style={{
                background: isCompleted
                  ? "linear-gradient(135deg, #d1fae5 0%, #ffffff 100%)"
                  : isCurrent
                  ? "linear-gradient(135deg, #dbeafe 0%, #ffffff 100%)"
                  : "linear-gradient(135deg, #f3f4f6 0%, #ffffff 100%)"
              }}
            >
              {isCompleted ? "✅" : isCurrent ? "🏛️" : "📋"}
            </motion.div>

            {/* 名称 */}
            <h3 className={cn(
              "text-center font-semibold text-lg mb-2",
              isCompleted
                ? "text-green-700"
                : isCurrent
                ? "text-blue-700 group-hover:text-blue-600"
                : "text-gray-900 group-hover:text-blue-600"
            )}>
              {name}
            </h3>

            {/* 状态指示 */}
            {isCompleted && (
              <motion.p
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                className="text-center text-sm text-green-600"
              >
                已完成 ✓
              </motion.p>
            )}

            {isCurrent && (
              <motion.div
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                className="text-center text-sm text-blue-600"
              >
                <span className="inline-flex items-center gap-1">
                  <span className="relative flex h-2 w-2">
                    <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-blue-400 opacity-75"></span>
                    <span className="relative inline-flex rounded-full h-2 w-2 bg-blue-500"></span>
                  </span>
                  待评价
                </span>
              </motion.div>
            )}

            {!isCompleted && !isCurrent && (
              <p className="text-center text-sm text-gray-400">
                等待评价
              </p>
            )}
          </div>

          {/* 底部箭头 */}
          <motion.div
            className="absolute bottom-4 right-4 text-gray-300 group-hover:text-blue-400 transition-colors"
            whileHover={{ x: 4 }}
          >
            <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 5l7 7-7 7" />
            </svg>
          </motion.div>
        </motion.div>
      </Link>
    </motion.div>
  )
}
```

**Step 2: Commit**

```bash
git add components/office-card.tsx
git commit -m "feat: enhance office card with immersive design and animations"
```

---

### Task 5: 重构首页为沉浸式布局

**Files:**
- Modify: `app/page.tsx`

**Step 1: Replace with new immersive design**

```typescript
// app/page.tsx
"use client"

import { useEffect, useState } from "react"
import { AnimatePresence, motion } from "framer-motion"
import Link from "next/link"
import { OfficeCard } from "@/components/office-card"
import { ProgressBar } from "@/components/progress-bar"
import { Trophy, TrendingUp, Users } from "lucide-react"
import { getDeviceId } from "@/lib/device-id"
import { pageTransition } from "@/lib/animations"

interface Office {
  id: number
  name: string
  displayOrder: number
}

export default function HomePage() {
  const [offices, setOffices] = useState<Office[]>([])
  const [completedOfficeIds, setCompletedOfficeIds] = useState<Set<number>>(new Set())
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    async function fetchOffices() {
      try {
        const deviceId = getDeviceId()
        const response = await fetch("/api/offices", {
          headers: {
            "x-device-id": deviceId,
          },
        })
        if (!response.ok) {
          throw new Error("获取股室列表失败")
        }
        const data = await response.json()
        setOffices(data.offices)
        setCompletedOfficeIds(new Set(data.completedOfficeIds || []))
      } catch (err) {
        setError(err instanceof Error ? err.message : "加载失败")
      } finally {
        setLoading(false)
      }
    }

    fetchOffices()
  }, [])

  // 获取当前应该评价的股室（第一个未完成的）
  const currentOfficeIndex = offices.findIndex(o => !completedOfficeIds.has(o.id))
  const completedCount = completedOfficeIds.size
  const totalCount = offices.length
  const allCompleted = completedCount === totalCount && totalCount > 0

  return (
    <motion.main
      variants={pageTransition}
      initial="initial"
      animate="enter"
      exit="exit"
      className="min-h-screen bg-gradient-to-b from-blue-50 via-white to-white"
    >
      <div className="max-w-5xl mx-auto px-4 py-6 md:py-8">
        {/* 头部 */}
        <header className="text-center py-6 md:py-8">
          <motion.div
            initial={{ y: -20, opacity: 0 }}
            animate={{ y: 0, opacity: 1 }}
            transition={{ duration: 0.5 }}
          >
            <h1 className="text-2xl md:text-3xl font-bold text-gray-900 mb-2">
              政府股室德能勤绩评价系统
            </h1>
            <p className="text-gray-600">请对以下股室进行评价</p>
          </motion.div>
        </header>

        {/* 加载状态 */}
        {loading ? (
          <div className="flex flex-col items-center justify-center py-20">
            <motion.div
              animate={{ rotate: 360 }}
              transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
              className="w-12 h-12 border-4 border-blue-600 border-t-transparent rounded-full"
            />
            <p className="mt-4 text-gray-500">加载中...</p>
          </div>
        ) : error ? (
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            className="text-center py-20"
          >
            <div className="text-6xl mb-4">⚠️</div>
            <p className="text-red-600 text-lg">{error}</p>
          </motion.div>
        ) : allCompleted ? (
          /* 全部完成状态 */
          <motion.div
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            transition={{ duration: 0.5 }}
            className="text-center py-16"
          >
            <motion.div
              animate={{
                y: [0, -10, 0],
                rotate: [0, 5, -5, 0]
              }}
              transition={{
                duration: 1,
                repeat: Infinity,
                repeatDelay: 0.5
              }}
              className="text-8xl mb-6"
            >
              🎉
            </motion.div>
            <h2 className="text-2xl font-bold text-gray-900 mb-3">
              感谢您的参与！
            </h2>
            <p className="text-gray-600 mb-8">您今天已完成所有评价</p>
            <Link
              href="/results"
              className="inline-flex items-center gap-2 px-8 py-3 bg-blue-600 text-white rounded-xl hover:bg-blue-700 transition-colors shadow-lg shadow-blue-200"
            >
              <Trophy className="w-5 h-5" />
              查看统计结果
            </Link>
          </motion.div>
        ) : (
          /* 主要内容区 */
          <AnimatePresence mode="wait">
            <motion.div
              key="main-content"
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
            >
              {/* 进度条 */}
              <motion.div
                initial={{ y: 20, opacity: 0 }}
                animate={{ y: 0, opacity: 1 }}
                className="mb-8"
              >
                <ProgressBar
                  current={completedCount}
                  total={totalCount}
                  showLabel={true}
                />
              </motion.div>

              {/* 卡片网格 */}
              <motion.div
                layout
                className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6"
              >
                <AnimatePresence mode="popLayout">
                  {offices.map((office, index) => {
                    const isCompleted = completedOfficeIds.has(office.id)
                    const isCurrent = index === currentOfficeIndex

                    return (
                      <OfficeCard
                        key={office.id}
                        id={office.id}
                        name={office.name}
                        index={index}
                        status={isCompleted ? "completed" : isCurrent ? "current" : "pending"}
                        totalOffices={totalCount}
                        completedCount={completedCount}
                      />
                    )
                  })}
                </AnimatePresence>
              </motion.div>

              {/* 底部操作区 */}
              <motion.div
                initial={{ y: 20, opacity: 0 }}
                animate={{ y: 0, opacity: 1 }}
                transition={{ delay: 0.3 }}
                className="mt-10 flex flex-col sm:flex-row gap-4 justify-center items-center"
              >
                <Link
                  href="/results"
                  className="inline-flex items-center gap-2 px-6 py-3 bg-white border-2 border-gray-200 text-gray-700 rounded-xl hover:border-blue-300 hover:text-blue-600 transition-all shadow-sm"
                >
                  <TrendingUp className="w-5 h-5" />
                  查看统计结果
                </Link>
              </motion.div>
            </motion.div>
          </AnimatePresence>
        )}
      </div>
    </motion.main>
  )
}
```

**Step 2: Commit**

```bash
git add app/page.tsx
git commit -m "feat: redesign homepage with immersive card layout and progress tracking"
```

---

### Task 6: 更新 API 返回已完成评价ID列表

**Files:**
- Modify: `app/api/offices/route.ts`

**Step 1: Read the current API route**

Run: `cat app/api/offices/route.ts`

**Step 2: Modify to include completed office IDs**

The API should return an additional field `completedOfficeIds: number[]` alongside `offices: Office[]`.

```typescript
// 在现有返回数据中添加
{
  offices: offices,
  completedOfficeIds: completedIds // 从数据库查询当前设备今日已评价的股室ID列表
}
```

**Step 3: Commit**

```bash
git add app/api/offices/route.ts
git commit -m "feat: add completedOfficeIds to offices API response"
```

---

## Phase 3: 评价页面交互升级

### Task 7: 创建评价进度组件

**Files:**
- Create: `components/evaluation-progress.tsx`

**Step 1: Write the component**

```typescript
// components/evaluation-progress.tsx
"use client"

import { motion } from "framer-motion"
import { Check } from "lucide-react"
import { cn } from "@/lib/utils"

interface EvaluationProgressProps {
  dimensions: { key: string; label: string; completed: boolean }[]
  className?: string
}

export function EvaluationProgress({ dimensions, className }: EvaluationProgressProps) {
  const completedCount = dimensions.filter(d => d.completed).length
  const totalCount = dimensions.length

  return (
    <div className={cn("space-y-3", className)}>
      <div className="flex justify-between items-center">
        <span className="text-sm font-medium text-gray-700">评价进度</span>
        <span className="text-sm text-gray-500">{completedCount}/{totalCount}</span>
      </div>

      {/* 进度条 */}
      <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
        <motion.div
          className="h-full bg-gradient-to-r from-blue-500 to-blue-600 rounded-full"
          initial={{ width: 0 }}
          animate={{ width: `${(completedCount / totalCount) * 100}%` }}
          transition={{ duration: 0.5, ease: "easeOut" }}
        />
      </div>

      {/* 维度指示器 */}
      <div className="flex gap-2 justify-center">
        {dimensions.map((dim, index) => (
          <motion.div
            key={dim.key}
            initial={{ scale: 0 }}
            animate={{ scale: 1 }}
            transition={{ delay: index * 0.05 }}
            className={cn(
              "w-8 h-8 rounded-full flex items-center justify-center text-xs font-medium transition-colors",
              dim.completed
                ? "bg-blue-600 text-white"
                : "bg-gray-200 text-gray-400"
            )}
          >
            {dim.completed ? (
              <Check className="w-4 h-4" />
            ) : (
              <span>{index + 1}</span>
            )}
          </motion.div>
        ))}
      </div>
    </div>
  )
}
```

**Step 2: Commit**

```bash
git add components/evaluation-progress.tsx
git commit -m "feat: add evaluation progress indicator component"
```

---

### Task 8: 创建评价选项卡片组件

**Files:**
- Create: `components\rating-option-card.tsx`

**Step 1: Write the component**

```typescript
// components/rating-option-card.tsx
"use client"

import { motion } from "framer-motion"
import { cn } from "@/lib/utils"

interface RatingOptionCardProps {
  value: string
  label: string
  selected: boolean
  onSelect: () => void
  index: number
}

const optionColors = {
  "优秀": {
    bg: "bg-green-50",
    border: "border-green-200",
    selectedBg: "bg-green-100",
    selectedBorder: "border-green-500",
    text: "text-green-700"
  },
  "较好": {
    bg: "bg-blue-50",
    border: "border-blue-200",
    selectedBg: "bg-blue-100",
    selectedBorder: "border-blue-500",
    text: "text-blue-700"
  },
  "一般": {
    bg: "bg-amber-50",
    border: "border-amber-200",
    selectedBg: "bg-amber-100",
    selectedBorder: "border-amber-500",
    text: "text-amber-700"
  },
  "差": {
    bg: "bg-red-50",
    border: "border-red-200",
    selectedBg: "bg-red-100",
    selectedBorder: "border-red-500",
    text: "text-red-700"
  }
}

export function RatingOptionCard({ value, label, selected, onSelect, index }: RatingOptionCardProps) {
  const colors = optionColors[value as keyof typeof optionColors] || optionColors["较好"]

  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ delay: index * 0.05 }}
      className="flex-1 min-w-0"
    >
      <motion.button
        whileHover={{ scale: selected ? 1 : 1.02 }}
        whileTap={{ scale: 0.98 }}
        onClick={onSelect}
        className={cn(
          "w-full relative overflow-hidden rounded-xl border-2 p-4 text-center transition-all cursor-pointer",
          colors.bg,
          colors.border,
          selected && (colors.selectedBg + " " + colors.selectedBorder + " shadow-md"),
          !selected && "hover:shadow-sm"
        )}
      >
        {/* 选中时的光晕效果 */}
        {selected && (
          <motion.div
            layoutId="selected-glow"
            className="absolute inset-0 bg-gradient-to-br from-white/50 to-transparent"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            transition={{ duration: 0.3 }}
          />
        )}

        {/* 内容 */}
        <span className={cn(
          "relative text-lg font-semibold",
          colors.text,
          selected && "scale-110 inline-block"
        )}>
          {label}
        </span>

        {/* 选中标记 */}
        {selected && (
          <motion.div
            initial={{ scale: 0 }}
            animate={{ scale: 1 }}
            className="absolute -top-2 -right-2 w-6 h-6 rounded-full bg-blue-600 text-white flex items-center justify-center text-xs"
          >
            <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={3} d="M5 13l4 4L19 7" />
            </svg>
          </motion.div>
        )}
      </motion.button>
    </motion.div>
  )
}
```

**Step 2: Commit**

```bash
git add components/rating-option-card.tsx
git commit -m "feat: add animated rating option card component"
```

---

### Task 9: 重构评价页面

**Files:**
- Modify: `app/evaluate/[id]/page.tsx`

**Step 1: Replace with enhanced version**

```typescript
// app/evaluate/[id]/page.tsx
"use client"

import { useEffect, useState, useTransition } from "react"
import { useRouter } from "next/navigation"
import { motion, AnimatePresence } from "framer-motion"
import { Button } from "@/components/ui/button"
import { RatingOptionCard } from "@/components/rating-option-card"
import { EvaluationProgress } from "@/components/evaluation-progress"
import { DIMENSION_DESCRIPTIONS, RATING_OPTIONS } from "@/lib/constants"
import { getDeviceId } from "@/lib/device-id"
import { pageTransition } from "@/lib/animations"
import { ChevronLeft, CheckCircle2 } from "lucide-react"

interface Office {
  id: number
  name: string
}

export default function EvaluatePage({ params }: { params: { id: string } }) {
  const router = useRouter()
  const [isPending, startTransition] = useTransition()
  const [office, setOffice] = useState<Office | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const [submitError, setSubmitError] = useState<string | null>(null)
  const [showSuccess, setShowSuccess] = useState(false)
  const [ratings, setRatings] = useState({
    virtue: "",
    ability: "",
    diligence: "",
    performance: "",
  })

  const officeId = parseInt(params.id)

  useEffect(() => {
    async function fetchOffice() {
      try {
        const response = await fetch(`/api/offices/${officeId}`)
        if (!response.ok) {
          if (response.status === 404) {
            setError("股室不存在")
          } else {
            throw new Error("获取股室信息失败")
          }
          return
        }
        const data = await response.json()
        setOffice(data)
      } catch (err) {
        setError(err instanceof Error ? err.message : "加载失败")
      } finally {
        setLoading(false)
      }
    }

    fetchOffice()
  }, [officeId])

  const handleRatingChange = (dimension: string, value: string) => {
    setRatings((prev) => ({ ...prev, [dimension]: value }))
    setSubmitError(null)
  }

  const handleSubmit = async () => {
    // 验证所有维度已填写
    if (Object.values(ratings).some((v) => !v)) {
      setSubmitError("请填写所有评价维度")
      return
    }

    startTransition(async () => {
      try {
        const deviceId = getDeviceId()
        const response = await fetch("/api/evaluations", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "x-device-id": deviceId,
          },
          body: JSON.stringify({
            office_id: officeId,
            ...ratings,
          }),
        })

        const data = await response.json()

        if (!response.ok) {
          setSubmitError(data.message || "提交失败")
          return
        }

        // 显示成功动画
        setShowSuccess(true)

        // 延迟后返回首页
        setTimeout(() => {
          router.push("/")
        }, 1500)
      } catch (err) {
        setSubmitError("网络错误，请重试")
      }
    })
  }

  const dimensionKeys = Object.keys(DIMENSION_DESCRIPTIONS) as Array<keyof typeof DIMENSION_DESCRIPTIONS>
  const dimensions = dimensionKeys.map(key => ({
    key,
    label: DIMENSION_DESCRIPTIONS[key],
    completed: !!ratings[key]
  }))
  const allFilled = Object.values(ratings).every((v) => v)
  const progress = dimensions.filter(d => d.completed).length

  if (loading) {
    return (
      <motion.main
        variants={pageTransition}
        initial="initial"
        animate="enter"
        className="min-h-screen bg-gradient-to-b from-blue-50 to-white flex items-center justify-center"
      >
        <motion.div
          animate={{ rotate: 360 }}
          transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
          className="w-12 h-12 border-4 border-blue-600 border-t-transparent rounded-full"
        />
      </motion.main>
    )
  }

  if (error || !office) {
    return (
      <motion.main
        variants={pageTransition}
        initial="initial"
        animate="enter"
        className="min-h-screen bg-gradient-to-b from-blue-50 to-white flex items-center justify-center px-4"
      >
        <div className="text-center">
          <div className="text-6xl mb-4">⚠️</div>
          <p className="text-red-600 mb-4 text-lg">{error || "股室不存在"}</p>
          <Button onClick={() => router.push("/")}>返回首页</Button>
        </div>
      </motion.main>
    )
  }

  return (
    <motion.main
      variants={pageTransition}
      initial="initial"
      animate="enter"
      exit="exit"
      className="min-h-screen bg-gradient-to-b from-blue-50 to-white"
    >
      <div className="max-w-2xl mx-auto px-4 py-6 md:py-8">
        {/* 头部 */}
        <motion.header
          initial={{ y: -20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          className="text-center py-6 md:py-8"
        >
          <motion.button
            onClick={() => router.push("/")}
            className="absolute top-6 left-4 p-2 rounded-lg hover:bg-white/50 transition-colors"
            whileHover={{ scale: 1.05 }}
            whileTap={{ scale: 0.95 }}
          >
            <ChevronLeft className="w-6 h-6 text-gray-600" />
          </motion.button>

          <div className="inline-block px-6 py-2 bg-blue-100 text-blue-700 rounded-full text-sm font-medium mb-4">
            正在评价
          </div>

          <h1 className="text-2xl md:text-3xl font-bold text-gray-900 mb-2">
            {office.name}
          </h1>
          <p className="text-gray-600">请完成以下评价</p>
        </motion.header>

        {/* 进度指示器 */}
        <motion.div
          initial={{ y: 20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          transition={{ delay: 0.1 }}
          className="mb-8"
        >
          <EvaluationProgress dimensions={dimensions} />
        </motion.div>

        {/* 评价卡片 */}
        <motion.div
          initial={{ y: 20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          transition={{ delay: 0.2 }}
          className="bg-white rounded-2xl shadow-lg border border-gray-100 overflow-hidden"
        >
          <div className="p-6 md:p-8 space-y-8">
            <AnimatePresence mode="wait">
              {dimensionKeys.map((key, dimensionIndex) => {
                const description = DIMENSION_DESCRIPTIONS[key]
                const value = ratings[key]

                return (
                  <motion.div
                    key={key}
                    layout
                    initial={{ opacity: 0, x: -20 }}
                    animate={{ opacity: 1, x: 0 }}
                    transition={{ delay: dimensionIndex * 0.1 }}
                  >
                    <label className="block text-base md:text-lg font-semibold text-gray-800 mb-4">
                      {description}
                    </label>

                    <div className="flex gap-3">
                      {RATING_OPTIONS.map((option, optionIndex) => (
                        <RatingOptionCard
                          key={option}
                          value={option}
                          label={option}
                          selected={value === option}
                          onSelect={() => handleRatingChange(key, option)}
                          index={optionIndex}
                        />
                      ))}
                    </div>
                  </motion.div>
                )
              })}
            </AnimatePresence>

            {/* 错误提示 */}
            <AnimatePresence>
              {submitError && (
                <motion.div
                  initial={{ opacity: 0, y: -10 }}
                  animate={{ opacity: 1, y: 0 }}
                  exit={{ opacity: 0, y: -10 }}
                  className="p-4 bg-red-50 border-2 border-red-200 rounded-xl text-red-600"
                >
                  {submitError}
                </motion.div>
              )}
            </AnimatePresence>

            {/* 提交按钮区 */}
            <div className="flex gap-4 pt-4 border-t border-gray-100">
              <Button
                variant="outline"
                className="flex-1 h-12 text-base"
                onClick={() => router.push("/")}
                disabled={isPending}
              >
                返回
              </Button>
              <Button
                className={cn(
                  "flex-1 h-12 text-base",
                  allFilled && "shadow-lg shadow-blue-200"
                )}
                onClick={handleSubmit}
                disabled={isPending || !allFilled}
              >
                {isPending ? (
                  <span className="flex items-center gap-2">
                    <motion.span
                      animate={{ rotate: 360 }}
                      transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
                    >
                      ⏳
                    </motion.span>
                    提交中...
                  </span>
                ) : (
                  <span className="flex items-center gap-2">
                    <CheckCircle2 className="w-5 h-5" />
                    提交评价
                  </span>
                )}
              </Button>
            </div>
          </div>
        </motion.div>

        {/* 成功弹窗 */}
        <AnimatePresence>
          {showSuccess && (
            <motion.div
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
              className="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4"
              onClick={() => setShowSuccess(false)}
            >
              <motion.div
                initial={{ scale: 0.5, opacity: 0 }}
                animate={{ scale: 1, opacity: 1 }}
                exit={{ scale: 0.5, opacity: 0 }}
                transition={{ type: "spring", damping: 15 }}
                className="bg-white rounded-3xl p-8 max-w-sm w-full text-center shadow-2xl"
                onClick={(e) => e.stopPropagation()}
              >
                <motion.div
                  initial={{ scale: 0, rotate: -180 }}
                  animate={{ scale: 1, rotate: 0 }}
                  transition={{ delay: 0.2, type: "spring" }}
                  className="text-6xl mb-4"
                >
                  🎉
                </motion.div>
                <h2 className="text-2xl font-bold text-gray-900 mb-2">
                  提交成功！
                </h2>
                <p className="text-gray-600 mb-6">感谢您的评价</p>
                <motion.div
                  animate={{ opacity: [0.5, 1, 0.5] }}
                  transition={{ duration: 1.5, repeat: Infinity }}
                  className="text-sm text-gray-400"
                >
                  正在返回首页...
                </motion.div>
              </motion.div>
            </motion.div>
          )}
        </AnimatePresence>
      </div>
    </motion.main>
  )
}
```

**Step 2: Commit**

```bash
git add app/evaluate/[id]/page.tsx
git commit -m "feat: redesign evaluation page with immersive UI and animations"
```

---

## Phase 4: 完成庆祝动画

### Task 10: 创建纸屑庆祝组件

**Files:**
- Create: `components/confetti-celebration.tsx`

**Step 1: Write the component**

```typescript
// components/confetti-celebration.tsx
"use client"

import { motion, AnimatePresence } from "framer-motion"
import { useEffect, useState } from "react"

interface ConfettiPieceProps {
  delay: number
  color: string
  x: number
}

function ConfettiPiece({ delay, color, x }: ConfettiPieceProps) {
  return (
    <motion.div
      initial={{ y: -100, x, opacity: 0, rotate: 0 }}
      animate={{
        y: ["0vh", "100vh"],
        opacity: [1, 1, 0],
        rotate: [0, 360, 720]
      }}
      transition={{
        delay,
        duration: 3,
        ease: "easeInOut"
      }}
      className="fixed w-3 h-3 rounded-sm z-50"
      style={{ backgroundColor: color, left: `${x}%` }}
    />
  )
}

const colors = [
  "#3b82f6", // blue
  "#10b981", // green
  "#f59e0b", // amber
  "#ef4444", // red
  "#8b5cf6", // purple
  "#ec4899", // pink
]

export function ConfettiCelebration({ active }: { active: boolean }) {
  const [pieces, setPieces] = useState<number>(0)

  useEffect(() => {
    if (active) {
      setPieces(50)
      const timer = setTimeout(() => setPieces(0), 4000)
      return () => clearTimeout(timer)
    }
  }, [active])

  return (
    <AnimatePresence>
      {active && (
        <div className="fixed inset-0 pointer-events-none z-50 overflow-hidden">
          {Array.from({ length: pieces }).map((_, i) => (
            <ConfettiPiece
              key={i}
              delay={Math.random() * 0.5}
              color={colors[Math.floor(Math.random() * colors.length)]}
              x={Math.random() * 100}
            />
          ))}
        </div>
      )}
    </AnimatePresence>
  )
}
```

**Step 2: Commit**

```bash
git add components/confetti-celebration.tsx
git commit -m "feat: add confetti celebration component"
```

---

### Task 11: 在首页添加庆祝动画

**Files:**
- Modify: `app/page.tsx`

**Step 1: Import and add ConfettiCelebration**

在文件顶部添加导入：
```typescript
import { ConfettiCelebration } from "@/components/confetti-celebration"
```

在组件内添加状态：
```typescript
const [showCelebration, setShowCelebration] = useState(false)
```

在 `useEffect` 中检测是否刚完成所有评价：
```typescript
useEffect(() => {
  // ... existing code ...
  if (completedCount === totalCount && totalCount > 0 && !loading) {
    setShowCelebration(true)
    const timer = setTimeout(() => setShowCelebration(false), 4000)
    return () => clearTimeout(timer)
  }
}, [completedCount, totalCount, loading])
```

在返回的 JSX 中添加组件（在 `</motion.main>` 之前）：
```typescript
<ConfettiCelebration active={showCelebration} />
```

**Step 2: Commit**

```bash
git add app/page.tsx
git commit -m "feat: add celebration animation when all evaluations completed"
```

---

## Phase 5: 移动端优化

### Task 12: 创建移动端触摸手势组件

**Files:**
- Create: `components/swable-card.tsx`

**Step 1: Write the component**

```typescript
// components/swipeable-card.tsx
"use client"

import { motion, useMotionValue, useTransform, PanInfo } from "framer-motion"
import { ReactNode } from "react"

interface SwipeableCardProps {
  children: ReactNode
  onSwipeLeft?: () => void
  onSwipeRight?: () => void
  className?: string
}

export function SwipeableCard({
  children,
  onSwipeLeft,
  onSwipeRight,
  className
}: SwipeableCardProps) {
  const x = useMotionValue(0)
  const rotate = useTransform(x, [-200, 200], [-15, 15])
  const opacity = useTransform(x, [-200, -100, 0, 100, 200], [0, 1, 1, 1, 0])

  const handleDragEnd = (_: any, info: PanInfo) => {
    const threshold = 100
    if (info.offset.x > threshold) {
      onSwipeRight?.()
    } else if (info.offset.x < -threshold) {
      onSwipeLeft?.()
    }
  }

  return (
    <motion.div
      style={{ x, rotate, opacity }}
      drag="x"
      dragConstraints={{ left: 0, right: 0 }}
      dragElastic={0.7}
      onDragEnd={handleDragEnd}
      className={className}
    >
      {children}
    </motion.div>
  )
}
```

**Step 2: Commit**

```bash
git add components/swipeable-card.tsx
git commit -m "feat: add swipeable card component for mobile gestures"
```

---

### Task 13: 添加移动端底部操作栏

**Files:**
- Create: `components/mobile-bottom-bar.tsx`

**Step 1: Write the component**

```typescript
// components/mobile-bottom-bar.tsx
"use client"

import { motion } from "framer-motion"
import { ChevronLeft, ChevronRight, BarChart3 } from "lucide-react"
import Link from "next/link"

interface MobileBottomBarProps {
  onPrevious?: () => void
  onNext?: () => void
  hasNext?: boolean
  hasPrevious?: boolean
}

export function MobileBottomBar({
  onPrevious,
  onNext,
  hasNext = true,
  hasPrevious = false
}: MobileBottomBarProps) {
  return (
    <motion.div
      initial={{ y: 100 }}
      animate={{ y: 0 }}
      className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 px-4 py-3 md:hidden safe-area-pb"
    >
      <div className="flex items-center justify-between max-w-lg mx-auto">
        <motion.button
          whileTap={{ scale: 0.95 }}
          onClick={onPrevious}
          disabled={!hasPrevious}
          className={cn(
            "p-3 rounded-xl",
            hasPrevious
              ? "bg-blue-100 text-blue-600"
              : "bg-gray-100 text-gray-400 cursor-not-allowed"
          )}
        >
          <ChevronLeft className="w-6 h-6" />
        </motion.button>

        <Link href="/results">
          <motion.button
            whileTap={{ scale: 0.95 }}
            className="p-3 rounded-xl bg-gray-100 text-gray-600"
          >
            <BarChart3 className="w-6 h-6" />
          </motion.button>
        </Link>

        <motion.button
          whileTap={{ scale: 0.95 }}
          onClick={onNext}
          disabled={!hasNext}
          className={cn(
            "p-3 rounded-xl",
            hasNext
              ? "bg-blue-600 text-white"
              : "bg-gray-100 text-gray-400 cursor-not-allowed"
          )}
        >
          <ChevronRight className="w-6 h-6" />
        </motion.button>
      </div>
    </motion.div>
  )
}
```

**Step 2: Commit**

```bash
git add components/mobile-bottom-bar.tsx
git commit -m "feat: add mobile bottom navigation bar"
```

---

### Task 14: 添加安全区域CSS

**Files:**
- Modify: `styles/globals.css`

**Step 1: Add safe area support**

在文件末尾添加：
```css
/* iOS 安全区域支持 */
@supports (padding: max(0px)) {
  .safe-area-pb {
    padding-bottom: max(1rem, env(safe-area-inset-bottom));
  }

  .safe-area-pt {
    padding-top: max(1rem, env(safe-area-inset-top));
  }
}

/* 移动端优化 */
@media (max-width: 768px) {
  /* 禁用双击缩放 */
  * {
    touch-action: manipulation;
  }

  /* 防止长按选中文本 */
  .no-select {
    -webkit-user-select: none;
    user-select: none;
  }
}

/* 暗色模式支持（预留） */
@media (prefers-color-scheme: dark) {
  /* 待实现 */
}
```

**Step 2: Commit**

```bash
git add styles/globals.css
git commit -m "feat: add safe area support and mobile optimizations"
```

---

## Phase 6: 测试与验证

### Task 15: 测试动画和交互

**Step 1: Start development server**

```bash
npm run dev
```

Expected: Server running on `http://127.0.0.1:8080`

**Step 2: Manual testing checklist**

访问 `http://127.0.0.1:8080` 并验证：

- [ ] 首页卡片依次淡入动画
- [ ] 进度条从0动画到当前值
- [ ] 鼠标悬停卡片时轻微放大和阴影效果
- [ ] 点击卡片进入评价页面，有过渡动画
- [ ] 评价页选项卡片选中时有弹性放大
- [ ] 评价进度指示器实时更新
- [ ] 提交成功后显示庆祝弹窗
- [ ] 完成所有评价后触发纸屑动画
- [ ] 移动端底部操作栏正确显示
- [ ] 所有动画流畅不卡顿

**Step 3: Browser console check**

打开浏览器开发者工具 Console，确认无错误：
```javascript
// 应该没有以下类型的错误：
// - framer-motion 相关错误
// - React hydration 错误
// - TypeScript 类型错误
```

**Step 4: Performance check**

```javascript
// 在 Console 中运行：
performance.mark('start')
// 进行一些页面操作
performance.mark('end')
performance.measure('interaction', 'start', 'end')
performance.getEntriesByName('interaction')[0].duration
```

Expected: 动画帧率保持 60fps

**Step 5: Commit if no issues**

```bash
git add .
git commit -m "test: verify animations and interactions working correctly"
```

---

## Phase 7: 可选增强功能

### Task 16 (Optional): 添加暗色模式支持

**Files:**
- Create: `components/theme-provider.tsx`
- Create: `components/theme-toggle.tsx`

**Step 1: Create theme provider**

```typescript
// components/theme-provider.tsx
"use client"

import { createContext, useContext, useEffect, useState } from "react"

type Theme = "light" | "dark"

const ThemeContext = createContext<{
  theme: Theme
  toggleTheme: () => void
}>({
  theme: "light",
  toggleTheme: () => {}
})

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light")

  useEffect(() => {
    const stored = localStorage.getItem("theme") as Theme | null
    if (stored) setTheme(stored)
    else if (window.matchMedia("(prefers-color-scheme: dark)").matches) {
      setTheme("dark")
    }
  }, [])

  useEffect(() => {
    document.documentElement.classList.toggle("dark", theme === "dark")
    localStorage.setItem("theme", theme)
  }, [theme])

  const toggleTheme = () => setTheme(t => t === "light" ? "dark" : "light")

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => useContext(ThemeContext)
```

**Step 2: Create toggle component**

```typescript
// components/theme-toggle.tsx
"use client"

import { motion } from "framer-motion"
import { Moon, Sun } from "lucide-react"
import { useTheme } from "@/components/theme-provider"

export function ThemeToggle() {
  const { theme, toggleTheme } = useTheme()

  return (
    <motion.button
      whileHover={{ scale: 1.05 }}
      whileTap={{ scale: 0.95 }}
      onClick={toggleTheme}
      className="p-2 rounded-lg bg-gray-200 dark:bg-gray-700"
    >
      <motion.div
        key={theme}
        initial={{ rotate: -180, opacity: 0 }}
        animate={{ rotate: 0, opacity: 1 }}
        transition={{ duration: 0.3 }}
      >
        {theme === "light" ? (
          <Sun className="w-5 h-5 text-amber-500" />
        ) : (
          <Moon className="w-5 h-5 text-blue-300" />
        )}
      </motion.div>
    </motion.button>
  )
}
```

**Step 3: Update tailwind.config**

```javascript
// tailwind.config.js - add darkMode config
module.exports = {
  darkMode: 'class',
  // ... rest of config
}
```

**Step 4: Commit**

```bash
git add components/theme-provider.tsx components/theme-toggle.tsx tailwind.config.js
git commit -m "feat: add dark mode support (optional)"
```

---

### Task 17 (Optional): 添加语音输入支持

**Files:**
- Create: `components/voice-input.tsx`

**Step 1: Create voice input component**

```typescript
// components/voice-input.tsx
"use client"

import { useState, useEffect } from "react"
import { motion } from "framer-motion"
import { Mic, MicOff, Volume2 } from "lucide-react"

interface VoiceInputProps {
  onResult: (text: string) => void
  language?: string
}

export function VoiceInput({ onResult, language = "zh-CN" }: VoiceInputProps) {
  const [listening, setListening] = useState(false)
  const [supported, setSupported] = useState(true)

  useEffect(() => {
    if (!("webkitSpeechRecognition" in window) && !("SpeechRecognition" in window)) {
      setSupported(false)
    }
  }, [])

  const toggleListening = () => {
    if (!supported) return

    const SpeechRecognition = (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition

    if (!SpeechRecognition) return

    const recognition = new SpeechRecognition()
    recognition.lang = language
    recognition.continuous = false
    recognition.interimResults = false

    recognition.onstart = () => setListening(true)
    recognition.onend = () => setListening(false)
    recognition.onresult = (event: any) => {
      const transcript = event.results[0][0].transcript
      onResult(transcript)
    }

    recognition.onerror = () => setListening(false)

    recognition.start()
  }

  if (!supported) return null

  return (
    <motion.button
      whileHover={{ scale: 1.05 }}
      whileTap={{ scale: 0.95 }}
      onClick={toggleListening}
      className={cn(
        "p-4 rounded-full transition-colors",
        listening
          ? "bg-red-100 text-red-600 animate-pulse"
          : "bg-blue-100 text-blue-600 hover:bg-blue-200"
      )}
    >
      {listening ? (
        <MicOff className="w-6 h-6" />
      ) : (
        <Mic className="w-6 h-6" />
      )}
    </motion.button>
  )
}
```

**Step 2: Commit**

```bash
git add components/voice-input.tsx
git commit -m "feat: add voice input component (optional)"
```

---

## Final Review and Documentation

### Task 18: 更新项目文档

**Files:**
- Create: `docs/FEATURES.md`
- Modify: `README.md`

**Step 1: Create features documentation**

```markdown
# 沉浸式评价交互系统 - 功能特性

## 核心特性

### 1. 沉浸式卡片流首页
- 卡片依次淡入动画
- 悬停放大和阴影效果
- 完成/待评价状态视觉区分
- 实时进度追踪

### 2. 流畅的评价体验
- 进度条实时反馈
- 选项卡片弹性动画
- 提交成功庆祝弹窗
- 自动返回首页

### 3. 完成庆祝
- 纸屑飞舞动画
- 感谢文案展示
- 成就感强化

### 4. 移动端优化
- 响应式布局
- 触摸手势支持
- 底部操作栏
- iOS 安全区域适配

### 5. 可选功能
- 暗色模式切换
- 语音输入支持

## 技术实现

- **动画引擎**: Framer Motion
- **样式**: Tailwind CSS
- **框架**: Next.js 14 App Router
- **类型安全**: TypeScript
```

**Step 2: Update README with new features**

```markdown
# 政府股室德能勤绩评价系统

一个基于 Next.js 的政府股室评价系统，采用沉浸式交互设计。

## 功能特性

- ✨ 沉浸式卡片流交互
- 📊 实时进度追踪
- 🎉 完成庆祝动画
- 📱 移动端优化
- 🎨 流畅动画体验

## 开发

```bash
npm install
npm run dev
```

访问 http://localhost:8080

## 技术栈

- Next.js 14
- React 18
- Tailwind CSS
- Framer Motion
- TypeScript
- Prisma
```

**Step 3: Commit**

```bash
git add docs/ README.md
git commit -m "docs: update documentation with new features"
```

---

### Task 19: 最终测试和清理

**Step 1: Run linter**

```bash
npm run lint
```

Expected: No linting errors

**Step 2: Type check**

```bash
npx tsc --noEmit
```

Expected: No type errors

**Step 3: Build test**

```bash
npm run build
```

Expected: Build succeeds without errors

**Step 4: Clean up any console.logs**

Search and remove any debug console.log statements:
```bash
grep -r "console.log" app/ components/ --exclude-dir=node_modules
```

**Step 5: Final commit**

```bash
git add .
git commit -m "chore: final cleanup and polish - immersive evaluation UI complete"
```

---

## Summary

This plan implements a complete UI overhaul of the evaluation system with:

1. **Phase 1**: Animation foundation (Framer Motion setup, progress bars, status icons)
2. **Phase 2**: Immersive homepage (card animations, progress tracking, celebration)
3. **Phase 3**: Enhanced evaluation page (progress indicators, animated options, success modal)
4. **Phase 4**: Celebration effects (confetti animation)
5. **Phase 5**: Mobile optimizations (touch gestures, bottom bar, safe areas)
6. **Phase 6**: Testing and validation
7. **Phase 7**: Optional enhancements (dark mode, voice input)

**Total Estimated Tasks**: 19
**Priority Tasks**: 1-15 (Core features)
**Optional Tasks**: 16-17 (Dark mode, voice input)
