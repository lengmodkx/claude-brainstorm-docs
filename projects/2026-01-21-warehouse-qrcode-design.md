# 仓库二维码库存查询功能设计

> 创建日期：2026-01-21
> 作者：Claude Code
> 状态：设计完成，待开发

## 1. 需求概述

### 1.1 功能描述
为 ERP 模块的每个仓库自动生成专属二维码，扫描二维码可查看当前仓库的库存信息（库存总览、商品明细、预警信息、出入库记录）。

### 1.2 需求总结
| 维度 | 要求 |
|------|------|
| 目标模块 | ERP 模块（ErpWarehouse） |
| 访问方式 | 公开访问（无需登录） |
| 展示内容 | 库存总览、商品明细、预警信息、出入库记录 |
| 访问终端 | 移动网页 + 微信小程序 |
| 二维码生成 | 管理员确认后手动生成 |

## 2. 整体架构

### 2.1 系统架构

采用前后端分离架构：

**后端扩展**（`yudao-module-erp`）：
- 新增 `warehouse-qrcode` 子模块
- 新增数据库表 `erp_warehouse_qrcode`
- 新增 API 接口提供 H5 页面数据查询

**前端扩展**（`front/hc_vue`）：
- 管理端：新增仓库二维码管理页面
- H5 端：新增移动端库存查看页面（响应式设计）

**微信小程序**（后期扩展）：
- 复用相同的 API 获取数据

### 2.2 数据流向

```
管理员操作 → 触发生成二维码 → 保存二维码配置 → 下载二维码图片
                                                    ↓
用户扫码 → 识别 URL → 打开 H5 页面 → 调用 API → 展示库存信息
```

### 2.3 技术方案（混合方案）

- H5 移动网页：扫码后在浏览器打开移动端页面
- 微信小程序：扫码后跳转到小程序（后期扩展）

## 3. 数据库设计

### 3.1 新增表：erp_warehouse_qrcode

```sql
CREATE TABLE `erp_warehouse_qrcode` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `warehouse_id` bigint NOT NULL COMMENT '仓库ID，关联 erp_warehouse.id',
  `qrcode_url` varchar(500) NOT NULL COMMENT 'H5访问URL',
  `qrcode_image_url` varchar(500) DEFAULT NULL COMMENT '二维码图片URL（OSS存储）',
  `miniapp_path` varchar(200) DEFAULT NULL COMMENT '小程序页面路径',
  `status` tinyint NOT NULL DEFAULT '1' COMMENT '状态：1-启用，0-禁用',
  `generated_by` bigint NOT NULL COMMENT '生成人工号',
  `generated_time` datetime NOT NULL COMMENT '生成时间',
  `last_scanned_time` datetime DEFAULT NULL COMMENT '最后扫码时间',
  `scan_count` int NOT NULL DEFAULT '0' COMMENT '扫码次数统计',
  `creator` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_warehouse_id` (`warehouse_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB COMMENT='仓库二维码配置表';
```

### 3.2 新增表：erp_warehouse_stock_summary（缓存统计）

```sql
CREATE TABLE `erp_warehouse_stock_summary` (
  `warehouse_id` bigint NOT NULL COMMENT '仓库ID',
  `product_count` int NOT NULL DEFAULT '0' COMMENT '商品种类数',
  `total_quantity` decimal(20,4) NOT NULL DEFAULT '0' COMMENT '库存总数量',
  `warning_count` int NOT NULL DEFAULT '0' COMMENT '预警商品数',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`warehouse_id`)
) ENGINE=InnoDB COMMENT='仓库库存汇总统计表';
```

## 4. 后端 API 设计

### 4.1 管理端 API（需要登录）

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 生成二维码 | POST | `/admin-api/erp/warehouse-qrcode/generate` | 管理员确认后生成 |
| 查询配置 | GET | `/admin-api/erp/warehouse-qrcode/{warehouseId}` | 查询二维码配置 |
| 批量下载 | GET | `/admin-api/erp/warehouse-qrcode/download-batch` | 批量下载二维码图片 |
| 更新状态 | PUT | `/admin-api/erp/warehouse-qrcode/{id}/status` | 禁用/启用二维码 |

### 4.2 H5 公开 API（无需登录）

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 库存总览 | GET | `/api/warehouse-stock/{shortCode}/summary` | 获取库存汇总信息 |
| 商品明细 | GET | `/api/warehouse-stock/{shortCode}/products` | 获取商品库存列表 |
| 预警信息 | GET | `/api/warehouse-stock/{shortCode}/warnings` | 获取预警商品列表 |
| 出入库记录 | GET | `/api/warehouse-stock/{shortCode}/records` | 获取出入库历史 |

### 4.3 响应示例

**库存总览响应：**
```json
{
  "warehouseName": "危化品仓库A",
  "warehouseAddress": "1号教学楼101室",
  "productCount": 150,
  "totalQuantity": 12500.00,
  "warningCount": 3,
  "lastUpdateTime": "2026-01-21 15:30:00"
}
```

## 5. 前端设计

### 5.1 管理端页面

**仓库列表扩展：**
- 新增"二维码"操作列
- 按钮：生成二维码、查看/下载、重新生成

**二维码预览弹窗：**
- 显示二维码图片
- 显示访问地址、生成时间、扫码次数
- 支持下载 PNG、批量下载

### 5.2 H5 移动端页面

**页面结构：**
1. 仓库信息头部（名称、地址）
2. Tab 切换（总览、明细、预警、记录）
3. 数据展示区域（响应式布局）

**技术栈：**
- Vue 3 + Vant UI（移动端组件库）
- 响应式设计，适配移动端

## 6. 核心实现

### 6.1 二维码生成流程

```
1. 管理员点击"生成二维码"
2. 后端生成唯一短码（8位 Base62）
3. 构建 H5 访问 URL
4. 使用 ZXing 库生成 QR Code 图片
5. 上传二维码图片到 OSS
6. 保存配置到数据库
7. 返回结果给前端
```

### 6.2 短码生成算法

```java
public class ShortCodeGenerator {
    private static final String ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final int CODE_LENGTH = 8;

    public static String generate(Long warehouseId) {
        long timestamp = System.currentTimeMillis();
        int random = ThreadLocalRandom.current().nextInt(1000, 9999);
        long combined = warehouseId * 10000000000L + timestamp % 100000000L + random;

        StringBuilder code = new StringBuilder();
        while (combined > 0 && code.length() < CODE_LENGTH) {
            code.append(ALPHABET.charAt((int)(combined % 62)));
            combined /= 62;
        }

        return StringUtils.leftPad(code.toString(), CODE_LENGTH, '0');
    }
}
```

### 6.3 缓存策略

- Redis 缓存库存汇总数据，TTL 5分钟
- 库存变更时自动刷新缓存
- 公开 API 实施限流保护（每 IP 每分钟 60 次）

## 7. 错误处理

### 7.1 异常场景

| 场景 | 处理方式 |
|------|----------|
| 二维码已禁用 | 显示友好提示页面 |
| 仓库不存在 | 返回 404 页面 |
| 网络异常 | 显示重试按钮 |
| 数据为空 | 显示"暂无数据" |

### 7.2 安全措施

- 短码 8 位随机字符（防遍历）
- 请求限流保护
- 数据脱敏（不显示敏感信息）
- 访问日志审计

## 8. 测试策略

### 8.1 测试用例

- 二维码生成唯一性测试
- 短码编解码测试
- 缓存过期测试
- API 异常场景测试
- H5 移动端兼容性测试

### 8.2 测试范围

- iOS Safari
- Android Chrome
- 微信内置浏览器
- 弱网环境测试

## 9. 部署方案

### 9.1 部署结构

```
后端：yudao-module-erp
  - Controller: ErpWarehouseQrcodeController
  - Service: WarehouseQrcodeService
  - DAL: ErpWarehouseQrcodeMapper
  - DO: ErpWarehouseQrcodeDO

前端：
  - 管理端：front/hc_vue/views/erp/warehouse/qrcode/
  - H5端：front/h5（独立项目）
```

### 9.2 Nginx 配置

```nginx
server {
    listen 443 ssl;
    server_name h5.yourdomain.com;

    location /w/ {
        root /var/www/h5;
        try_files $uri $uri/ /index.html;
    }

    location /api/warehouse-stock/ {
        proxy_pass http://backend:48080;
    }
}
```

## 10. 实施计划

| 阶段 | 任务 | 时间 |
|------|------|------|
| 1 | 数据库设计与创建 | 0.5天 |
| 2 | 后端 API 开发 | 2天 |
| 3 | 管理端前端开发 | 1.5天 |
| 4 | H5 页面开发 | 2天 |
| 5 | 联调测试 | 1天 |
| 6 | 部署上线 | 0.5天 |
| **合计** | | **7.5天** |

## 11. 后续扩展

- 微信小程序开发
- 数据分析（扫码热力图）
- 消息推送（库存预警通知）
- 批量导入生成二维码
