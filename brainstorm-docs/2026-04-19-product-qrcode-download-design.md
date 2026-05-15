# 物料二维码下载功能设计

## 1. 需求概述

在 `/erp/product/productInfo` 页面的列表中，增加二维码下载功能，支持单个下载和批量下载。

二维码中包含以下信息：
- `id` - 物料 ID
- `name` - 物料名称
- `barCode` - 物料条码（可能为空）
- `categoryId` - 物料分类 ID
- `category` - 物料分类名称
- `unitId` - 单位 ID
- `unit` - 单位名称

## 2. 二维码内容格式

```json
{"id":1024,"name":"浓硫酸","barCode":"2024001","categoryId":1,"category":"危险化学品","unitId":2,"unit":"毫升"}
```

**说明：**
- 使用 JSON 格式，便于扫码枪解析
- 包含完整 ID 信息，便于系统关联
- 字段名称简短，减少二维码数据量
- `barCode` 可能为空，JSON 中输出为 `null`
- `category`、`unit` 为空（null 或空白字符串）时，JSON 中输出为 `null`
- **所有文本字段超长时截取前 N 字符 + "..."**（见下表），截断后仍为有效字符串

**⚠️ 扫码端兼容性要求：**
- 扫码端解析 JSON 时必须能正确处理 `null` 值
- 推荐使用 JSON 解析库自动处理，而非字符串分割
- 示例（JavaScript）：`JSON.parse(content).name ?? ''`

**⚠️ 内容长度控制：**
- 二维码数据量直接影响识别成功率，任何文本字段过长都会导致二维码密度过高
- `name`、`category`、`unit` 统一截断规则：超过最大长度时截取前 N 字符 + "..."
- 系统扫码解析后，以 `id` 为准从数据库拉取完整信息

| 字段 | 最大长度 | 说明 |
|------|----------|------|
| `name` | 30 | 物料名称通常最长 |
| `category` | 20 | 分类名称 |
| `unit` | 10 | 单位名称较短 |

## 3. 技术方案

### 3.1 技术选型

- **二维码生成库**：使用项目中已有的 ZXing 3.5.2
  - `com.google.zxing:core`（核心库）
  - `com.google.zxing:javase`（`MatrixToImageWriter` 所在包，**需确认已引入**）
- **前端**：Element Plus 按钮组件
- **存储方式**：Base64 存数据库

### 3.2 存储方案

二维码图片转为 Base64 字符串，存储在产品表的 `qr_code` 字段中。

**优点：**
- 数据一致性有保障，物料记录和二维码绑定
- 运维简单，无需管理文件目录
- 分布式部署友好
- 删除物料时自动清理

**缺点与应对：**
- Base64 比原图大约 33%，一个 300x300 PNG 二维码约 5~10KB，Base64 后约 7~14KB
- 万级物料量下会增加表宽度，但单条数据量可控
- **列表查询时不返回 `qrCode` 字段**（见 5.8 节），只在详情接口返回，避免网络传输大文本

### 3.3 二维码生成规格

| 属性 | 值 | 说明 |
|------|-----|------|
| 尺寸 | 300 x 300 px | 兼顾显示清晰度和文件大小 |
| 容错级别 | M（15%） | 工业环境扫码建议至少 M 级，兼顾密度和容错 |
| 边距（margin）| 2 | 默认边距，保证扫描识别率 |
| 格式 | PNG | 无损压缩，透明背景 |

### 3.4 异步任务线程池配置

**问题：** Spring 默认 `@Async` 使用单线程执行器，大批量任务会排队阻塞。

**解决方案：** 配置自定义线程池 `QrCodeTaskExecutor`：

```java
@Bean("qrCodeTaskExecutor")
public TaskExecutor qrCodeTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);        // 核心线程数
    executor.setMaxPoolSize(8);         // 最大线程数
    executor.setQueueCapacity(500);     // 队列容量
    executor.setKeepAliveSeconds(60);   // 空闲线程存活时间
    executor.setThreadNamePrefix("qrcode-");
    executor.setRejectedExecutionHandler(new ThreadPoolTaskExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

**使用方式：**
```java
@Async("qrCodeTaskExecutor")
public void asyncBatchGenerateQrCode(List<Long> productIds) { ... }
```

## 4. 表结构变更

### 4.1 产品表新增字段

```sql
ALTER TABLE erp_product ADD COLUMN qr_code TEXT COMMENT '二维码(Base64)';
```

### 4.2 实体类变更

`ProductDO` 新增字段：
```java
private String qrCode; // 二维码(Base64)
```

`ProductRespVO` 新增字段：
```java
@ApiModelProperty("二维码(Base64)")
private String qrCode;

@ApiModelProperty("是否有二维码")
private Boolean hasQrCode;
```

**说明：** 列表接口只返回 `hasQrCode`（布尔值），不返回 `qrCode`（Base64 大文本），详情接口才返回完整 `qrCode`。

## 5. 后端设计

### 5.1 文件结构

```
yudao-module-erp/
├── controller/admin/product/
│   └── ProductController.java         # 新增重新生成、下载、预览、扫码解析接口
├── service/product/
│   ├── ProductService.java           # 业务方法（不变）
│   └── ProductQrCodeService.java    # 新增：二维码生成、异步任务
├── dal/dataobject/product/
│   └── ProductDO.java                # 新增 qrCode 字段
├── config/
│   └── QrCodeAsyncConfig.java        # 新增：异步线程池配置
└── framework/
    ├── QrCodeUtil.java               # 新增：二维码工具类
    └── QrCodeSigner.java             # 新增：二维码签名校验（可选安全增强）
```

### 5.2 Service 层新增/修改方法

```java
// ProductQrCodeService.java
// 新增：生成并保存二维码
void generateAndSaveQrCode(Long productId);

// 新增：批量生成并保存二维码（同步）
void batchGenerateAndSaveQrCode(List<Long> productIds);

// 新增：重新生成单个二维码
void regenerateProductQrCode(Long id);

// 新增：批量重新生成二维码（每条独立事务，失败不影响其他）
void batchRegenerateProductQrCode(List<Long> ids);

// 新增：异步批量生成二维码（用于导入等大批量场景）
@Async("qrCodeTaskExecutor")
void asyncBatchGenerateQrCode(List<Long> productIds);

// 新增：批量下载二维码，返回 ZIP 字节数组
byte[] batchDownloadQrCode(List<Long> ids);

// 新增：扫码解析二维码
ProductRespVO scanQrCode(String qrcodeContent);
```

### 5.3 Controller 层新增接口

```java
@PostMapping("/regenerate-qrcode")
@Operation(summary = "重新生成单个物料二维码")
@PreAuthorize("@ss.hasPermission('erp:product:update')")
public CommonResult<Boolean> regenerateProductQrCode(@RequestParam("id") Long id);

@PostMapping("/regenerate-qrcode-batch")
@Operation(summary = "批量重新生成物料二维码")
@PreAuthorize("@ss.hasPermission('erp:product:update')")
public CommonResult<Boolean> regenerateProductQrCodeBatch(@RequestBody List<Long> ids) {
    if (CollUtil.isEmpty(ids)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR, "ID 列表不能为空");
    }

    // 过滤 null 值并去重
    List<Long> validIds = ids.stream()
        .filter(Objects::nonNull)
        .distinct()
        .collect(Collectors.toList());

    // 数量校验：单次最多 200 个
    if (validIds.size() > 200) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR,
            "单次最多重新生成 200 个，已选 " + validIds.size() + " 个");
    }

    productQrCodeService.batchRegenerateProductQrCode(validIds);
    return success(true);
}

@GetMapping("/get-qrcode")
@Operation(summary = "获取物料二维码（用于预览）")
@PreAuthorize("@ss.hasPermission('erp:product:query')")
public CommonResult<String> getProductQrCode(@RequestParam("id") Long id) {
    ProductDO product = productService.getProduct(id);
    if (product == null) {
        throw new ServiceException(ErrorCodeConstants.PRODUCT_NOT_EXISTS);
    }
    // qrCode 为 null 或空字符串时，直接返回（前端判断 !res.data 即可）
    return success(product.getQrCode());
}

@GetMapping("/download-qrcode")
@Operation(summary = "下载单个物料二维码")
@PreAuthorize("@ss.hasPermission('erp:product:query')")
public ResponseEntity<byte[]> downloadQrCode(@RequestParam("id") Long id) {
    ProductDO product = productService.getProduct(id);
    if (product == null) {
        throw new ServiceException(ErrorCodeConstants.PRODUCT_NOT_EXISTS);
    }
    if (StrUtil.isBlank(product.getQrCode())) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_NOT_EXISTS, "该物料暂无二维码，请先重新生成");
    }

    String base64Data = product.getQrCode().replaceFirst("^data:[^;]+;base64,", "");
    byte[] imageBytes;
    try {
        imageBytes = Base64.getDecoder().decode(base64Data);
    } catch (IllegalArgumentException e) {
        log.error("[二维码下载] Base64 解码失败, productId={}", product.getId(), e);
        // 单个物料数据损坏时，直接抛异常，无需跳过
        throw new ServiceException(ErrorCodeConstants.QR_CODE_DOWNLOAD_ERROR, "二维码数据异常");
    }

    String safeName = QrCodeUtil.sanitizeFileName(product.getName());
    String safeBarCode = StrUtil.isNotBlank(product.getBarCode())
        ? QrCodeUtil.sanitizeFileName(product.getBarCode())
        : "无条码";
    String fileName = URLEncoder.encode(safeName + "_" + safeBarCode + ".png", StandardCharsets.UTF_8)
        .replaceAll("\\+", "%20");

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + fileName + "\"")
        .contentType(MediaType.IMAGE_PNG)
        .body(imageBytes);
}

@PostMapping("/download-qrcode-batch")
@Operation(summary = "批量下载物料二维码")
@PreAuthorize("@ss.hasPermission('erp:product:query')")
public ResponseEntity<byte[]> downloadQrCodeBatch(@RequestBody List<Long> ids) {
    if (CollUtil.isEmpty(ids)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR, "物料 ID 列表不能为空");
    }

    // 过滤 null 值并去重
    List<Long> validIds = ids.stream()
        .filter(Objects::nonNull)
        .distinct()
        .collect(Collectors.toList());

    // 数量校验：单次下载最多 500 个
    if (validIds.size() > 500) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR,
            "单次下载最多 500 个物料，已选 " + validIds.size() + " 个");
    }

    byte[] zipBytes = productQrCodeService.batchDownloadQrCode(validIds);

    String fileName = URLEncoder.encode(
        "物料二维码_" + System.currentTimeMillis() + ".zip",
        StandardCharsets.UTF_8
    ).replaceAll("\\+", "%20");

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + fileName + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(zipBytes);
}

@PostMapping("/scan-qrcode")
@Operation(summary = "扫码解析物料二维码")
@PreAuthorize("@ss.hasPermission('erp:product:query')")
public CommonResult<ProductRespVO> scanQrCode(@RequestBody String qrcodeContent) {
    // 兼容两种传参方式：
    // 方式1：{"content": "二维码JSON"} - 提取 content 字段
    // 方式2：纯文本字符串 "{"id":1024,...}" - 直接使用
    String content = qrcodeContent;
    try {
        Map<String, Object> map = JSON.parseObject(qrcodeContent, Map.class);
        if (map != null && map.containsKey("content")) {
            Object contentObj = map.get("content");
            // ⚠️ 不能用 String.valueOf(null)，它会返回 "null" 字符串（4个字母）
            content = contentObj != null ? contentObj.toString() : null;
        }
    } catch (Exception ignored) {
        // 不是 JSON 格式，直接使用原始字符串
    }
    
    if (StrUtil.isBlank(content)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_CONTENT_EMPTY, "二维码内容为空");
    }
    
    ProductRespVO product = productQrCodeService.scanQrCode(content);
    return success(product);
}
```

**说明：**
- 重新生成接口使用 `@RequestBody` 传递 ID 列表，避免 URL 长度限制（约 8KB）
- **下载接口使用 `ResponseEntity<byte[]>` 返回**，避免被全局响应包装器（如 `CommonResult` 自动包装）干扰，确保文件内容正确
- **中文文件名使用 `URLEncoder.encode` + `%20` 替换**，避免浏览器乱码
- **Base64 解码使用 `^data:[^;]+;base64,`**，匹配任意 MIME 类型，比 `\w+` 更健壮
- **扫码接口使用 `@RequestBody String`** 接收二维码内容，兼容纯文本和 `{"content": "..."}` 两种格式

### 5.4 创建/导入/更新时存储二维码

**核心原则：** Service 层保持原有 `@Transactional` 控制业务事务，二维码生成在 Controller 层事务外调用，避免 CPU 密集型操作占用数据库连接。

**创建时（Service 层 - 原有事务）：**
```java
@Transactional(rollbackFor = Exception.class)
public Long createProduct(ProductSaveReqVO createReqVO) {
    // ... 现有创建逻辑（只插入业务数据）...
    Long productId = ...;
    return productId;
}
```

**创建时（Controller 层 - 事务外生成二维码）：**
```java
@PostMapping("/create")
@Operation(summary = "创建物料")
@PreAuthorize("@ss.hasPermission('erp:product:create')")
public CommonResult<Long> createProduct(@RequestBody ProductSaveReqVO createReqVO) {
    // 1. 事务内创建业务数据
    Long productId = productService.createProduct(createReqVO);
    
    // 2. 事务外生成二维码（CPU 密集型，不占用数据库连接）
    try {
        productQrCodeService.generateAndSaveQrCode(productId);
    } catch (Exception e) {
        log.error("[二维码生成] 创建物料后生成二维码失败, productId={}", productId, e);
        // 二维码生成失败不影响已创建的业务数据，返回成功但提示前端
        return success(productId, "物料创建成功，二维码生成失败，请手动重新生成");
    }
    
    return success(productId);
}
```

**导入时（Service 层）：**
```java
@Autowired
private ProductQrCodeService productQrCodeService;

public ProductImportRespVO importProductList(List<ProductImportVO> importList, boolean updateSupport) {
    // ... 现有导入逻辑（在 @Transactional 内）...

    // 导入成功后，异步批量生成二维码（避免接口超时）
    // ProductQrCodeService 独立管理 @Async 方法，无自调用问题
    List<Long> productIds = ...;

    // ⚠️ 事务边界注意：
    // asyncBatchGenerateQrCode 在事务内被调用，如果导入事务回滚，异步任务可能仍会被执行。
    // 由于异步任务内部会重新查询数据库校验物料存在性，空跑不会导致脏数据，只会记录错误日志。
    // 如需严格保证一致性，可使用 TransactionSynchronizationManager.afterCommit()：
    // TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    //     @Override public void afterCommit() {
    //         productQrCodeService.asyncBatchGenerateQrCode(productIds);
    //     }
    // });
    productQrCodeService.asyncBatchGenerateQrCode(productIds);

    // 记录异步生成数量，前端可据此展示提示
    log.info("[物料导入] 成功, 共 {} 条, 异步生成二维码中", productIds.size());

    return result;
}
```

**导入时（Controller 层 - 如需返回提示）：**
如果前端需要知道有多少物料正在异步生成二维码，可在 `ProductImportRespVO` 中新增字段：
```java
@ApiModelProperty("正在异步生成二维码的数量")
private Integer qrCodeGeneratingCount;
```
前端导入成功后展示："成功导入 N 个物料，二维码正在后台生成，请稍后刷新查看"

**更新时（Service 层 - 原有事务）：**
```java
@Transactional(rollbackFor = Exception.class)
public Boolean updateProduct(ProductSaveReqVO updateReqVO) {
    // ... 现有更新逻辑（只更新业务数据）...
    return true;
}
```

**更新时（Controller 层 - 事务外生成二维码）：**
```java
@PutMapping("/update")
@Operation(summary = "更新物料")
@PreAuthorize("@ss.hasPermission('erp:product:update')")
public CommonResult<Boolean> updateProduct(@RequestBody ProductSaveReqVO updateReqVO) {
    // 1. 事务内更新业务数据
    Boolean success = productService.updateProduct(updateReqVO);
    if (!Boolean.TRUE.equals(success)) {
        throw new ServiceException(ErrorCodeConstants.PRODUCT_UPDATE_FAILED, "物料更新失败");
    }
    
    // 2. 事务外重新生成二维码（CPU 密集型，不占用数据库连接）
    try {
        productQrCodeService.regenerateProductQrCode(updateReqVO.getId());
    } catch (Exception e) {
        log.error("[二维码生成] 更新物料后重新生成二维码失败, productId={}", updateReqVO.getId(), e);
        // ⚠️ 如果项目 CommonResult 没有 success(T, String) 重载，
        //    可改为 return success(true); 并在前端通过消息提示展示失败信息
        return success(true, "物料更新成功，二维码重新生成失败，请手动重试");
    }
    
    return success(true);
}
```

**⚠️ 事务边界说明：**
- `createProduct` / `updateProduct` 保持原有 `@Transactional`，只负责业务数据更新
- **二维码生成在 Controller 层事务外调用**，避免 CPU 密集型操作拉长数据库连接持有时间
- `generateAndSaveQrCode` 内部使用 `UpdateWrapper` 只更新 `qr_code` 字段，避免触发其他自动填充逻辑
- 二维码生成失败不影响已提交的业务数据，前端会收到友好提示
- **二维码相关方法抽取到独立的 `ProductQrCodeService`**，避免 `@Async` 自调用问题，代码更清晰

### 5.5 QrCodeUtil 工具类

```java
/**
 * 二维码工具类
 * 使用项目中已有的 ZXing 库
 */
public class QrCodeUtil {

    // 工具类禁止实例化
    private QrCodeUtil() {
        throw new UnsupportedOperationException("工具类禁止实例化");
    }

    private static final int QR_CODE_WIDTH = 300;
    private static final int QR_CODE_HEIGHT = 300;

    /**
     * 生成二维码图片
     * @param content 二维码内容
     * @return BufferedImage
     */
    public static BufferedImage generateQrCode(String content) {
        return generateQrCode(content, QR_CODE_WIDTH, QR_CODE_HEIGHT);
    }

    /**
     * 生成二维码图片
     * @param content 二维码内容
     * @param width 宽度（像素）
     * @param height 高度（像素）
     * @return BufferedImage
     */
    public static BufferedImage generateQrCode(String content, int width, int height) {
        QRCodeWriter qrCodeWriter = new QRCodeWriter();
        HashMap<EncodeHintType, Object> hints = new HashMap<>();
        hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.M); // 容错级别 M
        hints.put(EncodeHintType.MARGIN, 2); // 边距
        hints.put(EncodeHintType.CHARACTER_SET, "UTF-8"); // 编码
        BitMatrix bitMatrix = qrCodeWriter.encode(content, BarcodeFormat.QR_CODE, width, height, hints);
        return MatrixToImageWriter.toBufferedImage(bitMatrix);
    }

    /**
     * 图片转 Base64
     * @param image BufferedImage
     * @return Base64 字符串（含 data URI scheme）
     */
    public static String toBase64(BufferedImage image) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            ImageIO.write(image, "PNG", baos);
            byte[] bytes = baos.toByteArray();
            return "data:image/png;base64," + Base64.getEncoder().encodeToString(bytes);
        } catch (IOException e) {
            throw new ServiceException(ErrorCodeConstants.QR_CODE_GENERATE_ERROR, "二维码 Base64 编码失败");
        }
    }

    /**
     * 生成物料二维码内容（JSON 格式，所有文本字段超长时自动截断）
     *
     * @param product 物料实体（含 id, name, barCode, categoryId, unitId）
     * @param categoryName 分类名称（从分类表查询）
     * @param unitName 单位名称（从单位表查询）
     * @return JSON 字符串
     */
    public static String buildProductQrCodeContent(ProductDO product, String categoryName, String unitName) {
        // 使用 LinkedHashMap 保证字段顺序固定，确保相同输入始终生成相同的二维码图片（幂等性）
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("id", product.getId());
        map.put("name", truncate(product.getName(), 30));
        // barCode 为 null 或空白字符串时统一输出 JSON null，与 2.2 节声明一致
        map.put("barCode", isBlank(product.getBarCode()) ? null : product.getBarCode());
        map.put("categoryId", product.getCategoryId());
        // 空字符串转为 null，保证 JSON 输出 null，与 2.2 节声明一致
        map.put("category", isBlank(categoryName) ? null : truncate(categoryName, 20));
        map.put("unitId", product.getUnitId());
        map.put("unit", isBlank(unitName) ? null : truncate(unitName, 10));
        // 使用项目中统一的 JSON 工具（如 FastJson/Hutool，根据项目实际选型）
        // ⚠️ 必须输出 null 值（如 category/unit 为空时），确保扫码端能正确判断字段存在性
        // FastJSON: JSON.toJSONString(map, SerializerFeature.WriteMapNullValue)
        // Hutool:   JSONUtil.toJsonStr(map)  // Hutool 默认输出 null
        return JSON.toJSONString(map);
    }

    /**
     * 文本截断工具方法
     */
    private static String truncate(String str, int maxLen) {
        if (str == null) return null;  // 保持 JSON null，与 2.2 节声明一致
        if (str.length() <= maxLen) return str;
        if (maxLen <= 3) return str.substring(0, maxLen); // 避免 maxLen-3 为负数
        return str.substring(0, maxLen - 3) + "..."; // 截断后总长度不超过 maxLen
    }

    /**
     * 判断字符串是否为空（null 或空白字符串）
     */
    private static boolean isBlank(String str) {
        return str == null || str.trim().isEmpty();
    }

    /**
     * 清理文件名字符串，移除 Windows 非法字符
     * Windows 非法字符：\ / : * ? " < > |
     */
    public static String sanitizeFileName(String name) {
        if (name == null) return "";
        // \\ 在 Java 字符串中解析为 \，正则中 \ 匹配反斜杠字符
        return name.replaceAll("[\\\\/:*?\"<>|]", "_");
    }
}
```

**⚠️ JSON 序列化库选型：**
- 文档中使用 `JSON.toJSONString()`（FastJson 风格）
- 如果项目使用 Hutool 的 `JSONUtil.toJsonStr()`，请替换为对应实现
- **注意保持与项目其他模块一致**

### 5.6 错误处理

| 场景 | 处理方式 |
|------|----------|
| 物料不存在 | 抛出 ServiceException |
| 物料 ID 不合法 | 参数校验失败 |
| Base64 编码失败 | 记录日志，抛异常 |
| 二维码生成失败 | 记录日志，抛异常（事务外，不影响业务数据） |
| 二维码不存在（下载/预览时）| 抛出 ServiceException，前端提示"暂无二维码" |
| 批量下载全空无二维码 | 抛出 ServiceException，提示"选中物料均暂无二维码" |
| 扫码内容格式错误 | 抛出 ServiceException，提示"二维码格式无效" |
| 扫码物料不存在 | 抛出 ServiceException，提示"物料不存在或已删除" |
| 扫码物料已禁用 | 抛出 ServiceException，提示"物料已禁用" |

### 5.7 异步任务失败处理（可选）

**问题：** 导入触发的异步任务如果失败（服务重启、异常等），部分物料可能没有二维码。

**简化方案（推荐）：**
不新增状态字段，采用"懒加载"策略：
- 列表接口的 `hasQrCode` 直接判断 `qr_code` 字段是否为空
- 前端下载/预览时如果 `hasQrCode = false`，提示用户并支持"一键重新生成"

**如需精细化跟踪（可选）：**
可新增 `qr_code_status` 字段跟踪生成状态，提供重试机制：

```sql
-- 可选：如需状态跟踪，新增状态字段
ALTER TABLE erp_product ADD COLUMN qr_code_status TINYINT DEFAULT 0 COMMENT '二维码状态：0-待生成 1-已生成 3-生成失败';
```

```java
// 可选：异步任务完成后更新状态
@Async("qrCodeTaskExecutor")
public void asyncBatchGenerateQrCode(List<Long> productIds) {
    if (CollUtil.isEmpty(productIds)) {
        return;
    }
    // 过滤 null 值并去重，避免 NPE 和重复处理
    List<Long> validIds = productIds.stream()
        .filter(Objects::nonNull)
        .distinct()
        .collect(Collectors.toList());
    for (Long productId : validIds) {
        try {
            generateAndSaveQrCode(productId);
            // 可选：updateQrCodeStatus(productId, QrCodeStatus.GENERATED);
        } catch (Exception e) {
            log.error("二维码生成失败, productId={}", productId, e);
            // 可选：updateQrCodeStatus(productId, QrCodeStatus.FAILED);
        }
    }
}
```

**⚠️ 串行处理说明：**
`asyncBatchGenerateQrCode` 内部使用 `for` 循环串行处理每个物料，这是有意设计：
- 二维码生成是 **CPU 密集型** 操作，并行处理多个二维码并不能提升速度（受 CPU 核心数限制）
- 串行处理避免瞬间耗尽线程池，保护系统稳定性
- 异步的价值在于**不阻塞主业务线程**，而非单个任务内部并行
- 每批 100 条在异步线程中串行执行，实测性能可接受（约 1~2 秒/百条）
- 如需进一步优化，可将 `for` 循环改为 `parallelStream()`（需评估线程池影响）

**⚠️ updateTime 不被修改说明：**
`generateAndSaveQrCode` 使用 `UpdateWrapper` 只更新 `qr_code` 字段：
```java
productMapper.update(null, new LambdaUpdateWrapper<ProductDO>()
    .eq(ProductDO::getId, productId)
    .set(ProductDO::getQrCode, base64));
```
- 传入的 entity 为 `null`，不会触发自动填充
- 只设置 `qr_code` 字段，其他字段不受影响
- 单元测试需验证：`generateAndSaveQrCode` 执行后 `updateTime` 保持不变

### 5.8 列表接口优化

**列表查询（productPage / productList）：**
- 不返回 `qrCode` 字段（Base64 大文本）
- 返回 `hasQrCode` 布尔值，前端据此判断是否显示下载/预览按钮

**详情查询（getProduct）：**
- 返回完整 `qrCode` 字段

```java
// ProductRespVO 转换时
respVO.setHasQrCode(StrUtil.isNotBlank(productDO.getQrCode()));
respVO.setQrCode(null); // 列表接口不返回，由查询详情的接口返回
```

### 5.9 批量下载实现（后端 ZIP 打包）

```java
public byte[] batchDownloadQrCode(List<Long> ids) {
    if (CollUtil.isEmpty(ids)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR, "物料 ID 列表不能为空");
    }

    // 过滤 null 值并去重，避免查询异常和重复打包
    List<Long> validIds = ids.stream()
        .filter(Objects::nonNull)
        .distinct()
        .collect(Collectors.toList());
    if (CollUtil.isEmpty(validIds)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR, "物料 ID 列表无效");
    }

    // 批量查询（减少数据库往返）
    List<ProductDO> products = productMapper.selectBatchIds(validIds);
    if (CollUtil.isEmpty(products)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_NOT_EXISTS, "物料不存在");
    }

    try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
         ZipOutputStream zos = new ZipOutputStream(baos)) {

        int successCount = 0;
        for (ProductDO product : products) {
            if (StrUtil.isBlank(product.getQrCode())) {
                continue; // 跳过无二维码的物料
            }

            // Base64 解码为图片字节（使用 [^;]+ 匹配任意 MIME 类型，更健壮）
            String base64Data = product.getQrCode().replaceFirst("^data:[^;]+;base64,", "");
            byte[] imageBytes;
            try {
                imageBytes = Base64.getDecoder().decode(base64Data);
            } catch (IllegalArgumentException e) {
                log.error("[二维码批量下载] Base64 解码失败, productId={}, 数据异常跳过", product.getId(), e);
                continue; // 跳过当前物料，继续处理其他
            }

            // ZIP 条目文件名：{id}_{物料名}_{条码}.png，用 id 保证唯一性
            String safeName = QrCodeUtil.sanitizeFileName(product.getName());
            String safeBarCode = StrUtil.isNotBlank(product.getBarCode())
                ? QrCodeUtil.sanitizeFileName(product.getBarCode())
                : "无条码";
            String baseFileName = product.getId() + "_" + safeName + "_" + safeBarCode + ".png";

            // ZIP 规范限制条目名不超过 200 字符，超长时截断但保留 id
            String fileName = baseFileName.length() > 200
                ? product.getId() + "_物料_" + baseFileName.substring(0, 180) + ".png"
                : baseFileName;

            ZipEntry entry = new ZipEntry(fileName);
            zos.putNextEntry(entry);
            zos.write(imageBytes);
            zos.closeEntry();
            successCount++;
        }

        zos.finish();

        // 如果没有任何二维码被加入 ZIP，抛出异常避免下载空文件
        if (successCount == 0) {
            throw new ServiceException(ErrorCodeConstants.QR_CODE_NOT_EXISTS,
                "选中的物料均暂无二维码，请稍后重试或重新生成");
        }

        return baos.toByteArray();
    } catch (IOException e) {
        log.error("批量下载二维码失败, ids={}", validIds, e);
        throw new ServiceException(ErrorCodeConstants.QR_CODE_DOWNLOAD_ERROR, "打包下载失败");
    }
}
```

### 5.10 generateAndSaveQrCode 实现

```java
@Service
public class ProductQrCodeService {
    
    @Autowired
    private ProductMapper productMapper;
    @Autowired
    private ErpProductCategoryMapper productCategoryMapper;  // 需注入分类 Mapper
    @Autowired
    private ErpProductUnitMapper productUnitMapper;          // 需注入单位 Mapper
    
    public void generateAndSaveQrCode(Long productId) {
        ProductDO product = productMapper.selectById(productId);
        if (product == null) {
            throw new ServiceException(ErrorCodeConstants.PRODUCT_NOT_EXISTS,
                "物料不存在，无法生成二维码");
        }
        
        // 查询分类名称和单位名称（ProductDO 通常只存外键 ID）
        String categoryName = "";
        String unitName = "";
        if (product.getCategoryId() != null) {
            ErpProductCategoryDO category = productCategoryMapper.selectById(product.getCategoryId());
            categoryName = category != null ? category.getName() : "";
        }
        if (product.getUnitId() != null) {
            ErpProductUnitDO unit = productUnitMapper.selectById(product.getUnitId());
            unitName = unit != null ? unit.getName() : "";
        }
        
        String qrCodeContent = QrCodeUtil.buildProductQrCodeContent(product, categoryName, unitName);
        BufferedImage image = QrCodeUtil.generateQrCode(qrCodeContent);
        String base64 = QrCodeUtil.toBase64(image);
        
        // 使用 UpdateWrapper 只更新 qr_code 字段，避免触发自动填充（如 updateTime）
        productMapper.update(null, new LambdaUpdateWrapper<ProductDO>()
            .eq(ProductDO::getId, productId)
            .set(ProductDO::getQrCode, base64));
        
        log.info("[二维码生成] 成功, productId={}", productId);
    }
}
```

### 5.11 扫码解析实现

```java
public ProductRespVO scanQrCode(String qrcodeContent) {
    if (StrUtil.isBlank(qrcodeContent)) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_PARAM_ERROR, "二维码内容不能为空");
    }
    
    try {
        // 1. 解析 JSON
        Map<String, Object> map = JSON.parseObject(qrcodeContent);
        Object idObj = map.get("id");
        if (idObj == null) {
            throw new ServiceException(ErrorCodeConstants.QR_CODE_FORMAT_ERROR, "二维码格式无效，缺少物料 ID");
        }
        Long productId;
        try {
            productId = Long.valueOf(idObj.toString());
        } catch (NumberFormatException e) {
            throw new ServiceException(ErrorCodeConstants.QR_CODE_FORMAT_ERROR, "二维码物料 ID 格式错误");
        }
        
        // 2. 验证签名（如果启用了签名）
        // String sign = (String) map.get("sign");
        // if (!qrCodeSigner.verify(qrcodeContent, sign)) {
        //     throw new ServiceException(ErrorCodeConstants.QR_CODE_SIGN_ERROR, "二维码签名验证失败");
        // }
        
        // 3. 查询物料详情（以数据库为准，不依赖二维码内容）
        ProductDO product = productMapper.selectById(productId);
        if (product == null) {
            throw new ServiceException(ErrorCodeConstants.PRODUCT_NOT_EXISTS, "物料不存在或已删除");
        }
        
        // 4. 校验物料状态（假设有 status 字段，0-启用 1-禁用）
        if (product.getStatus() != null && product.getStatus() == 1) {
            throw new ServiceException(ErrorCodeConstants.PRODUCT_DISABLED, "物料已禁用");
        }
        
        return ProductConvert.INSTANCE.convert(product);
    } catch (JSONException e) {
        throw new ServiceException(ErrorCodeConstants.QR_CODE_FORMAT_ERROR, "二维码内容不是有效的 JSON 格式");
    }
}
```

### 5.12 历史数据迁移

现有物料需要一次性补生成二维码，提供初始化脚本：

```java
/**
 * 历史数据二维码初始化
 * 通过配置控制是否执行，首次上线时开启，执行完成后关闭配置
 */
@Component
@ConditionalOnProperty(name = "qrcode.init.enabled", havingValue = "true")
public class ProductQrCodeInit implements CommandLineRunner {
    @Autowired
    private ProductQrCodeService productQrCodeService;
    @Autowired
    private ProductMapper productMapper;

    @Override
    public void run(String... args) {
        // 查询所有 qr_code 为空（null 或空字符串）的物料
        List<ProductDO> products = productMapper.selectList(
            new LambdaQueryWrapper<ProductDO>()
                .isNull(ProductDO::getQrCode)
                .or()
                .eq(ProductDO::getQrCode, "")
        );
        if (CollUtil.isEmpty(products)) {
            log.info("[二维码初始化] 无待处理物料");
            return;
        }

        List<Long> ids = products.stream().map(ProductDO::getId).collect(Collectors.toList());
        // 分批异步生成，每批 100 条（JDK 原生 subList，不依赖 Guava）
        for (int i = 0; i < ids.size(); i += 100) {
            // ⚠️ subList 返回原列表视图，必须 new ArrayList 创建副本再传给异步方法
            List<Long> batch = new ArrayList<>(ids.subList(i, Math.min(i + 100, ids.size())));
            productQrCodeService.asyncBatchGenerateQrCode(batch);
        }
        log.info("[二维码初始化] 共 {} 条物料待生成二维码", ids.size());
    }
}
```

**配置示例（`application.yaml`）：**
```yaml
qrcode:
  init:
    enabled: true  # 首次上线设为 true，执行完成后改回 false
```

**说明：**
- 通过 `@ConditionalOnProperty` 控制，防止重复执行
- 首次上线前开启配置，执行完成后立即关闭
- 分批异步处理，避免启动时阻塞
- 使用 JDK 原生 `List.subList()` 分批，不依赖 Guava 库
- **⚠️ 线程池压垮风险：** 如果历史数据量极大（如 10 万条 = 1000 批），`@Async` 线程池队列（500）可能被瞬间填满，
  `CallerRunsPolicy` 会让主线程执行剩余任务，导致应用启动被阻塞。建议：
  - 首次上线前手动执行一次性脚本（推荐）
  - 或在循环中增加 `Thread.sleep(500)` 给线程池消化时间
  - 或改为在独立后台线程中同步处理，不依赖 `@Async`

### 5.13 幂等性

`regenerateProductQrCode` 方法天然幂等：
- 无论调用多少次，只要物料信息不变，生成的二维码内容相同
- 即使多次调用，也只是重新生成相同的二维码图片覆盖原记录

**批量重新生成事务边界：**
- `batchRegenerateProductQrCode` **不加 `@Transactional`**，每条独立处理
- 单个物料生成失败只影响当前物料，其他物料继续处理
- 失败物料记录日志，前端可查看并单独重试

```java
@Service
public class ProductQrCodeService {
    
    // ... 注入的 Mapper ...
    
    /**
     * 重新生成单个物料二维码（幂等：相同输入 = 相同输出）
     * 直接复用 generateAndSaveQrCode，避免代码重复
     */
    public void regenerateProductQrCode(Long id) {
        generateAndSaveQrCode(id);
    }
    
    /**
     * 批量重新生成二维码（不加事务，每条独立处理）
     */
    public void batchRegenerateProductQrCode(List<Long> ids) {
        if (CollUtil.isEmpty(ids)) {
            return;
        }
        // 过滤 null 值并去重，避免 NPE 和重复处理
        List<Long> validIds = ids.stream()
            .filter(Objects::nonNull)
            .distinct()
            .collect(Collectors.toList());
        for (Long id : validIds) {
            try {
                regenerateProductQrCode(id);
            } catch (Exception e) {
                log.error("[二维码批量重生成] 失败, id={}", id, e);
            }
        }
    }
}
```

### 5.14 批量重新生成接口扩展设计

当前接口仅支持按 ID 列表重新生成，单次最多 200 个。为支持更灵活的业务场景（如按分类、按时间范围），预留扩展设计：

**方案 A（当前版本）：** 按 ID 列表
```java
@PostMapping("/regenerate-qrcode-batch")
public CommonResult<Boolean> regenerateProductQrCodeBatch(@RequestBody List<Long> ids) {
    // 单次最多 200 个，超出返回错误
}
```

**方案 B（未来扩展）：按条件重新生成**
```java
// 预留扩展接口，按条件重新生成
@PostMapping("/regenerate-qrcode-by-condition")
public CommonResult<Integer> regenerateProductQrCodeByCondition(
    @RequestBody QrCodeRegenerateCondVO cond  // 包含 categoryId、beginTime、endTime 等查询条件
);
```

**说明：** 当前版本实现方案 A，预留方案 B 接口扩展。

## 6. 前端设计

### 6.1 前端文件变更

```
front/hc_vue/src/
├── api/erp/product/product/index.ts    # 新增获取、下载、重新生成 API
└── views/erp/product/product/index.vue  # 操作列增加下载/预览/重生成按钮
```

### 6.2 Product 类型扩展

```typescript
export interface Product {
  // ... 现有字段 ...
  qrCode?: string      // 二维码(Base64)，详情接口返回
  hasQrCode?: boolean  // 是否有二维码，列表接口返回
}
```

### 6.3 按钮布局

**列表顶部工具栏：**
```vue
<el-button type="success" plain @click="handleBatchDownloadQrCode" :disabled="isEmpty(checkedIds)">
  <Icon icon="ep:download" class="mr-5px" /> 批量下载二维码
</el-button>
```

**每行操作列：**
```vue
<el-button link type="primary" @click="handleDownloadQrCode(scope.row)">
  下载
</el-button>
<el-button link type="info" @click="handlePreviewQrCode(scope.row)">
  预览
</el-button>
<el-button link type="warning" @click="handleRegenerateQrCode(scope.row.id)">
  重生成
</el-button>
```

### 6.4 重新生成二维码

```typescript
const handleRegenerateQrCode = async (id: number) => {
  try {
    const res = await ProductApi.regenerateQrCode(id)
    // 后端返回额外消息时展示（如"重新生成成功，二维码内容超出长度限制已自动截断"）
    if (res.msg && res.msg !== 'success') {
      message.warning(res.msg)
    } else {
      message.success('重新生成成功')
    }
    // 刷新列表更新 hasQrCode 状态
    getList()
  } catch (error) {
    message.error('重新生成失败：' + error.message)
  }
}
```

### 6.5 下载单个二维码（调后端接口）

**说明：** 列表接口不返回 `qrCode`，单个下载改为调后端 `download-qrcode` 接口，后端返回图片文件。

```typescript
// 清理文件名中的非法字符（与后端保持一致）
// Windows 非法字符：\ / : * ? " < > |
const sanitizeFileName = (name: string): string => {
  if (!name) return ''
  return name.replace(/[\\/:*?"<>|]/g, '_')
}

const handleDownloadQrCode = async (row: Product) => {
  if (!row.hasQrCode) {
    message.error('该物料暂无二维码，请稍后重试')
    return
  }
  
  const loading = ElLoading.service({ text: '正在下载...' })
  try {
    const response = await request.get({
      url: `/erp/product/download-qrcode?id=${row.id}`,
      responseType: 'blob'
    })
    
    // ⚠️ 如果 request 包装器返回的是 axios response 对象而非 raw data，
    //    需改为 new Blob([response.data], { type: 'image/png' })
    const blob = new Blob([response], { type: 'image/png' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    // 文件名：物料名_条码_时间戳.png（含毫秒，避免1秒内重复点击覆盖）
    // barCode 为空时用"无条码"代替，避免双下划线
    const timestamp = dayjs().format('YYYYMMDDHHmmssSSS')
    const safeName = sanitizeFileName(row.name)
    const safeBarCode = row.barCode ? sanitizeFileName(row.barCode) : '无条码'
    link.download = `${safeName}_${safeBarCode}_${timestamp}.png`
    link.click()
    URL.revokeObjectURL(url)
    message.success('下载成功')
  } catch (error) {
    message.error('下载失败：' + error.message)
  } finally {
    loading.close()
  }
}
```

### 6.6 批量下载二维码（调后端 ZIP 接口）

**说明：** 批量下载前先预检，提示用户有多少物料将被跳过。

```typescript
import { ElMessageBox } from 'element-plus'

const handleBatchDownloadQrCode = async () => {
  if (isEmpty(checkedIds.value)) {
    message.warning('请先选择物料')
    return
  }

  // 预检：统计无二维码的物料数量
  const selectedRows = list.value.filter(item => checkedIds.value.includes(item.id))
  const noQrCodeItems = selectedRows.filter(item => !item.hasQrCode)

  // loading 放在 try 外部，确保任何分支都会关闭
  const loading = ElLoading.service({ text: '正在打包下载...' })
  let userConfirmed = false

  try {
    if (noQrCodeItems.length > 0) {
      await ElMessageBox.confirm(
        `选中的 ${selectedRows.length} 个物料中，有 ${noQrCodeItems.length} 个暂无二维码，将被跳过。是否继续下载？`,
        '提示',
        { confirmButtonText: '继续下载', cancelButtonText: '取消', type: 'warning' }
      )
    }
    userConfirmed = true

    const response = await request.post({
      url: '/erp/product/download-qrcode-batch',
      data: checkedIds.value,
      responseType: 'blob'
    })

    // ⚠️ 如果 request 包装器返回的是 axios response 对象而非 raw data，
    //    需改为 new Blob([response.data], { type: 'application/zip' })
    const blob = new Blob([response], { type: 'application/zip' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = `物料二维码_${dayjs().format('YYYYMMDDHHmmss')}.zip`
    link.click()
    URL.revokeObjectURL(url)

    message.success('下载成功')
  } catch (error) {
    // ElMessageBox.confirm 取消 / 请求错误 / 其他异常统一处理
    message.error('下载失败或已取消')
  } finally {
    // 确保 loading 始终关闭，无论哪个分支
    loading.close()
  }
}
```

**优点：**
- 前端预检：下载前提示用户有多少物料被跳过，让用户决定是否继续
- 后端兜底：即使前端没做检查，后端也会跳过无二维码的物料并抛出明确错误
- 列表无需返回 `qrCode` 大文本，前端无需引入 JSZip

### 6.7 二维码预览（调后端接口获取 Base64）

**说明：** 列表接口不返回 `qrCode`，预览时先调 `get-qrcode` 接口获取 Base64。

```typescript
const previewVisible = ref(false)
const previewQrCode = ref('')
const previewTitle = ref('')

const handlePreviewQrCode = async (row: Product) => {
  if (!row.hasQrCode) {
    message.error('暂无二维码')
    return
  }
  
  const loading = ElLoading.service({ text: '正在加载...' })
  try {
    const res = await ProductApi.getQrCode(row.id)
    if (!res.data) {
      message.error('二维码加载失败')
      return
    }
    previewQrCode.value = res.data
    previewTitle.value = `${row.name} - 二维码`
    previewVisible.value = true
  } catch (error) {
    message.error('预览失败：' + error.message)
  } finally {
    loading.close()
  }
}
```

```vue
<el-dialog 
  v-model="previewVisible" 
  :title="previewTitle" 
  width="400px" 
  align-center
  @closed="previewQrCode = ''"
>
  <div style="text-align: center;">
    <img :src="previewQrCode" style="width: 300px; height: 300px;" />
    <p style="color: #999; font-size: 12px; margin-top: 10px;">请使用扫码枪或微信扫一扫识别</p>
  </div>
</el-dialog>
```

### 6.8 前端 API

```typescript
// 获取二维码（预览用）
getQrCode: async (id: number) => {
  return await request.get({ url: `/erp/product/get-qrcode?id=${id}` })
},

// 下载单个二维码（返回 blob）
downloadQrCode: async (id: number) => {
  return await request.get({
    url: `/erp/product/download-qrcode?id=${id}`,
    responseType: 'blob'
  })
},

// 重新生成单个二维码
regenerateQrCode: async (id: number) => {
  return await request.post({ url: `/erp/product/regenerate-qrcode?id=${id}` })
},

// 批量重新生成二维码（body 传参）
regenerateQrCodeBatch: async (ids: number[]) => {
  return await request.post({ url: '/erp/product/regenerate-qrcode-batch', data: ids })
},

// 批量下载二维码（返回 blob）
downloadQrCodeBatch: async (ids: number[]) => {
  return await request.post({
    url: '/erp/product/download-qrcode-batch',
    data: ids,
    responseType: 'blob'
  })
}
```

## 7. 扫码解析设计

### 7.1 扫码场景

二维码设计用于以下场景：
- **PDA/扫码枪扫描**：仓库出入库、盘点时扫描物料二维码快速识别
- **手机扫码**：移动端查看物料信息
- **PC 扫码枪**：实验室管理平台上扫码录入

### 7.2 扫码流程

```
用户扫码 → 获取 JSON 内容 → 调用 /scan-qrcode 接口 → 后端解析 JSON 提取 id 
→ 查询数据库获取完整物料信息 → 校验物料状态 → 返回物料详情
```

**关键原则：**
- 扫码后以 `id` 为准查询数据库，不依赖二维码中的名称等冗余信息（防止截断后的信息误导）
- 校验物料状态，已禁用的物料拒绝扫码
- 支持签名验证（可选），防止伪造二维码

### 7.3 扫码接口

见 5.3 节 `scanQrCode` 接口定义，5.11 节扫码解析实现。

### 7.4 扫码端集成建议

**PDA/移动端扫码页面示例：**

```typescript
// 移动端扫码后调用
const handleScanResult = async (scanResult: string) => {
  try {
    const res = await request.post({
      url: '/erp/product/scan-qrcode',
      data: scanResult  // 二维码 JSON 内容直接放 body 中
    })
    // 跳转到物料详情页或执行入库/盘点等业务操作
    router.push(`/product/detail/${res.data.id}`)
  } catch (error) {
    message.error('扫码失败：' + error.message)
  }
}
```

## 8. 安全设计

### 8.1 二维码内容安全

当前二维码包含物料内部信息（`id`、`categoryId` 等），存在以下风险：
- 二维码被拍照外传后，外部可获取系统内部 ID
- 二维码内容被篡改后伪造，可能误导系统

**应对措施（按需实现）：**
- **方案 A（简单）：** 二维码内容增加签名字段，系统扫码时校验签名合法性
- **方案 B（安全）：** 二维码内容整体 AES 加密，扫码后由系统解密解析（需配套扫码端改造）
- **当前版本建议：** 采用方案 A，增加 `sign` 字段

```java
@Component
public class QrCodeSigner {
    
    @Value("${qrcode.secret-key}")
    private String secretKey;
    
    public String sign(String content) {
        // Hutool: SecureUtil.hmacSha256(key).digestBase64(content)
        // JDK 原生: Mac.getInstance("HmacSHA256") + SecretKeySpec + Base64
        return SecureUtil.hmacSha256(secretKey).digestBase64(content);
    }
    
    public boolean verify(String content, String sign) {
        String expected = sign(content);
        return Objects.equals(expected, sign);
    }
}
```

```yaml
# application.yaml
qrcode:
  secret-key: your-secret-key-here  # 生产环境建议从配置中心读取
```

### 8.2 接口权限

- 重新生成接口需要 `erp:product:update` 权限
- 下载/预览/扫码接口需要 `erp:product:query` 权限
- 批量操作增加前端 loading，防止重复点击

## 9. 错误码定义

新增以下错误码（根据项目实际错误码定义方式调整）：

| 错误码 | 含义 | 使用场景 |
|--------|------|----------|
| QR_CODE_GENERATE_ERROR | 二维码生成失败 | Base64 编码、图片生成失败 |
| QR_CODE_DOWNLOAD_ERROR | 二维码下载失败 | ZIP 打包、文件写入失败 |
| QR_CODE_PARAM_ERROR | 二维码参数错误 | ID 为空、内容为空 |
| QR_CODE_NOT_EXISTS | 二维码不存在 | 下载/预览时物料无二维码 |
| QR_CODE_FORMAT_ERROR | 二维码格式错误 | 扫码内容不是有效 JSON、缺少 id |
| QR_CODE_SIGN_ERROR | 二维码签名错误 | 签名验证失败（如启用） |
| QR_CODE_CONTENT_EMPTY | 二维码内容为空 | 扫码接口 content 字段为空 |
| PRODUCT_NOT_EXISTS | 物料不存在 | 扫码时物料已删除 |
| PRODUCT_DISABLED | 物料已禁用 | 扫码时物料状态为禁用 |

## 10. 实现步骤

1. **后端：表结构变更**
   - 产品表新增 qr_code 字段

2. **后端：实体类**
   - ProductDO 新增 qrCode 字段
   - ProductRespVO 新增 qrCode、hasQrCode 字段

3. **后端：异步线程池配置**
   - 新增 QrCodeAsyncConfig.java，配置自定义线程池

4. **后端：工具类**
   - 创建 QrCodeUtil.java（含统一截断、文件名清理、签名生成）

5. **后端：Service 层**
   - 新增 ProductQrCodeService.java，实现二维码相关方法
   - 实现 generateAndSaveQrCode、batchGenerateAndSaveQrCode（使用 UpdateWrapper）
   - 实现 regenerateProductQrCode、batchRegenerateProductQrCode（使用 UpdateWrapper，批量不加事务）
   - 新增 asyncBatchGenerateQrCode 异步批量生成
   - 新增 batchDownloadQrCode 后端 ZIP 打包（含全空校验、文件名清理）
   - 新增 scanQrCode 扫码解析（含状态校验）
   - ProductService 修改 createProduct、importProductList、updateProduct 方法（只处理业务数据）

6. **后端：Controller 层**
   - 新增重新生成二维码接口（支持 body 传批量 ID，单次最多 200 个）
   - 新增获取二维码接口（get-qrcode，用于预览）
   - 新增单个下载二维码接口（download-qrcode，返回图片文件）
   - 新增批量下载二维码接口（download-qrcode-batch，返回 ZIP，单次最多 500 个）
   - 新增扫码解析接口（scan-qrcode，使用 @RequestBody，兼容两种格式）
   - 修改 create、update 接口，**事务外**调用二维码生成（失败时友好提示）

7. **后端：列表接口优化**
   - 列表不返回 qrCode，只返回 hasQrCode

8. **后端：历史数据迁移**
   - 新增 ProductQrCodeInit 初始化脚本（带配置开关）

9. **前端：API 层**
   - 新增 getQrCode、downloadQrCode、regenerateQrCode、regenerateQrCodeBatch、downloadQrCodeBatch 方法

10. **前端：页面组件**
    - 顶部工具栏增加批量下载按钮
    - 操作列增加预览、下载、重生成按钮
    - 实现预览弹窗（调 get-qrcode 接口，关闭时清空 Base64）
    - 实现单个下载逻辑（调 download-qrcode 接口，含文件名清理）
    - 实现批量下载逻辑（调 download-qrcode-batch 接口，含预检确认）

11. **移动端/PDA（如有）**
    - 集成扫码解析接口，扫码后跳转物料详情或执行业务操作

## 11. 单元测试

| 测试项 | 说明 |
|--------|------|
| QrCodeUtil.generateQrCode | 验证生成的二维码尺寸、格式正确 |
| QrCodeUtil.buildProductQrCodeContent | 验证 name/category/unit 超长时自动截断；验证空字符串转为 null |
| QrCodeUtil.sanitizeFileName | 验证 `\ / : * ? " < > \|` 都被替换为下划线 |
| QrCodeUtil.toBase64 | 验证 Base64 编码正确，包含 data URI scheme |
| QrCodeUtil（签名）| 验证签名生成与校验逻辑 |
| ProductQrCodeService.generateAndSaveQrCode | 验证生成后数据库记录正确（含分类/单位名称查询），updateTime 不被修改 |
| ProductQrCodeService.regenerateProductQrCode | 验证幂等性（多次调用结果一致），updateTime 不被修改 |
| ProductQrCodeService.batchRegenerateProductQrCode | 验证批量时单条失败不影响其他，不加事务 |
| ProductQrCodeService.asyncBatchGenerateQrCode | 验证异步任务执行、批量查询 |
| ProductQrCodeService.batchDownloadQrCode | 验证 ZIP 打包正确，跳过无二维码物料，全空时抛异常，文件名无非法字符；验证用 id 保证文件名唯一 |
| ProductQrCodeService.scanQrCode | 验证正常解析、无效 JSON、物料不存在、物料禁用等场景 |
| Controller 下载接口 | 验证 ResponseEntity 返回正确，中文文件名编码正常；验证数量超限时返回错误 |
| Controller 预览接口 | 验证返回 Base64 正确 |
| Controller 扫码接口 | 验证 @RequestBody 接收、兼容两种格式、权限控制、参数校验 |
| Controller 创建接口 | 验证二维码生成失败时返回友好提示（业务数据不受影响） |
| 事务边界测试 | 验证 create/update 事务内只处理业务数据，二维码在事务外生成 |
| @Async 自调用测试 | 验证 ProductQrCodeService 中 @Async 方法无自调用问题 |

### updateTime 不被修改的测试用例

```java
@Test
void generateAndSaveQrCode_shouldNotModifyUpdateTime() {
    // 准备：插入物料，记录原始 updateTime
    ProductDO product = new ProductDO();
    product.setName("测试物料");
    product.setCreateTime(new Date());
    product.setUpdateTime(originalUpdateTime);
    productMapper.insert(product);
    Long productId = product.getId();

    // 执行：生成二维码
    productQrCodeService.generateAndSaveQrCode(productId);

    // 验证：updateTime 不变，二维码已生成
    ProductDO updated = productMapper.selectById(productId);
    assertEquals(originalUpdateTime, updated.getUpdateTime());
    assertNotNull(updated.getQrCode());
}

@Test
void regenerateProductQrCode_shouldNotModifyUpdateTime() {
    // 准备：插入带二维码的物料
    ProductDO product = new ProductDO();
    product.setName("测试物料");
    product.setQrCode(existingQrCode);
    product.setUpdateTime(originalUpdateTime);
    productMapper.insert(product);
    Long productId = product.getId();

    // 执行：重新生成
    productQrCodeService.regenerateProductQrCode(productId);

    // 验证：updateTime 不变，二维码已重新生成
    ProductDO updated = productMapper.selectById(productId);
    assertEquals(originalUpdateTime, updated.getUpdateTime());
    assertNotEquals(existingQrCode, updated.getQrCode());
}
```

## 12. 验收标准

- [ ] 新建物料时自动生成二维码并存入数据库（二维码失败时提示手动重试，业务数据不受影响）
- [ ] 导入物料成功后异步批量生成二维码（ProductQrCodeService 独立管理 @Async）
- [ ] 更新物料信息后自动重新生成二维码（事务外执行，updateTime 不变）
- [ ] 列表页面顶部有批量下载按钮，操作列有单个下载/预览/重生成按钮
- [ ] 单个下载功能正常（后端返回图片文件，前端接收 blob）
- [ ] 批量下载功能正常（后端打包 ZIP，前端接收 blob 下载）
- [ ] 批量下载时前端预检提示跳过数量，后端自动跳过无二维码物料
- [ ] 批量下载全空无二维码时给出友好提示，不下载空文件
- [ ] 批量下载文件名用 id 保证唯一性，不会因同名覆盖
- [ ] 批量下载单次最多 500 个，超出返回友好错误
- [ ] 批量重新生成单次最多 200 个，超出返回友好错误
- [ ] 预览功能正常（调 get-qrcode 接口获取 Base64 显示，关闭时清空）
- [ ] 扫码解析功能正常（调 scan-qrcode 接口，以数据库为准，禁用的物料拒绝扫码）
- [ ] 重新生成二维码功能正常，支持幂等调用，不触发 updateTime 更新
- [ ] 二维码内容包含：id、name、barCode、categoryId、category、unitId、unit
- [ ] 二维码内容中空字符串转为 null，保证扫码端解析一致
- [ ] 历史物料上线后自动补生成二维码（通过配置控制）
- [ ] name/category/unit 超过最大长度时二维码内容自动截断，仍可正常扫码
- [ ] 异步任务使用独立线程池，不阻塞主业务
- [ ] 二维码生成在事务外执行，不占用数据库连接
- [ ] 前端批量下载无论成功、失败、取消，loading 均能正确关闭
