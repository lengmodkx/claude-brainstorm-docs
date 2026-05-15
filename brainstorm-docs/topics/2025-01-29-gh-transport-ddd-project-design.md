# 物流ERP系统DDD微服务架构设计

> 基于 2025-01-28-bff-architecture 架构图的微服务项目框架设计

> **重要提示**：本项目采用微服务架构，所有限界上下文独立部署，通过服务治理框架进行通信。

## 1. 项目概述

### 1.1 系统定位
物流ERP系统是一个综合性的企业级物流管理平台，采用 **DDD（领域驱动设计）+ BFF（Backend For Frontend）+ 微服务** 架构，支持多端接入（Web、Mobile、Mini Program、Admin Portal）。

### 1.2 技术栈总览

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| 前端 | Vue 3 / React 18 | Web端使用Vue 3，Admin端使用React |
| 移动端 | Flutter / React Native | 跨平台移动应用开发 |
| BFF层 | Spring Cloud Gateway | API网关，为各端定制API |
| 服务注册/配置 | Nacos | 服务注册发现、配置中心 |
| 限流熔断 | Sentinel | 流量控制、熔断降级 |
| 认证授权 | JWT + OAuth2 | 统一认证授权 |
| 领域层 | Spring Boot 3.x | DDD四层架构 |
| RPC调用 | OpenFeign | 服务间HTTP调用 |
| 事件驱动 | Kafka | 异步事件解耦 |
| 数据库 | PostgreSQL + MongoDB | 结构化数据 + 文档数据 |
| 缓存 | Redis | 会话缓存、分布式锁 |
| 分布式事务 | Seata | Saga模式 |

### 1.3 微服务列表

| 服务名 | 限界上下文 | 端口 | 数据库 | 说明 |
|--------|-----------|------|--------|------|
| gh-transport-gateway-web | Web BFF | 8080 | - | Web端API网关 |
| gh-transport-gateway-mobile | Mobile BFF | 8081 | - | Mobile端API网关 |
| gh-transport-gateway-admin | Admin BFF | 8082 | - | Admin端API网关 |
| gh-transport-auth | Auth BC | 8083 | PostgreSQL | 认证授权服务 |
| gh-transport-order | Order BC | 8084 | PostgreSQL | 订单管理服务 |
| gh-transport-dispatch | Dispatch BC | 8085 | PostgreSQL | 调度管理服务 |
| gh-transport-transport | Transport BC | 8086 | MongoDB | 运输管理服务 |
| gh-transport-customer | Customer BC | 8087 | PostgreSQL | 客户管理服务 |
| gh-transport-inventory | Inventory BC | 8088 | PostgreSQL | 库存管理服务 |

---

## 2. 项目结构

### 2.1 整体目录结构

```
gh-transport/
├── gh-transport-parent/                    # 父POM，统一版本管理
├── gh-transport-gateway/                   # BFF网关层
│   ├── gh-transport-gateway-web/          # Web BFF
│   ├── gh-transport-gateway-mobile/       # Mobile BFF
│   └── gh-transport-gateway-admin/        # Admin BFF
├── gh-transport-common/                   # 公共模块
│   ├── common-core/                       # 核心工具类
│   ├── common-redis/                      # Redis封装
│   ├── common-web/                        # Web通用组件
│   └── common-ddd/                        # DDD基类
├── gh-transport-auth/                     # 认证限界上下文
│   ├── auth-domain/                       # 领域层
│   ├── auth-application/                  # 应用层
│   ├── auth-infrastructure/               # 基础设施层
│   └── auth-interfaces/                   # 接口层
├── gh-transport-order/                    # 订单限界上下文
├── gh-transport-dispatch/                 # 调度限界上下文
├── gh-transport-transport/                # 运输限界上下文
├── gh-transport-customer/                 # 客户限界上下文
├── gh-transport-inventory/                # 库存限界上下文
├── gh-transport-event/                    # 事件总线模块
│   ├── event-core/                        # 事件核心
│   ├── event-kafka/                       # Kafka实现
│   └── event-rocketmq/                    # RocketMQ实现
└── gh-transport-front/                    # 前端项目
    ├── gh-transport-web/                  # Web端 (Vue 3)
    ├── gh-transport-admin/                # Admin端 (React)
    └── gh-transport-mobile/               # Mobile端 (Flutter)

```

### 2.2 限界上下文标准结构

以 `gh-transport-order` 为例：

```
gh-transport-order/
├── pom.xml                                # 模块POM
├── order-domain/                          # 领域层（核心）
│   ├── src/main/java/.../order/domain/
│   │   ├── aggregate/                     # 聚合根
│   │   │   └── Order.java
│   │   ├── entity/                        # 实体
│   │   │   └── OrderItem.java
│   │   ├── valueobject/                   # 值对象
│   │   │   ├── OrderNo.java
│   │   │   ├── OrderStatus.java
│   │   │   ├── Address.java
│   │   │   ├── Money.java
│   │   │   └── Quantity.java
│   │   ├── repository/                    # 仓储接口
│   │   │   └── OrderRepository.java
│   │   ├── service/                       # 领域服务
│   │   │   └── OrderDomainService.java
│   │   ├── event/                         # 领域事件
│   │   │   ├── OrderCreatedEvent.java
│   │   │   └── OrderCancelledEvent.java
│   │   └── exception/                     # 领域异常
│   │       └── OrderDomainException.java
│   └── pom.xml
├── order-application/                     # 应用层
│   ├── src/main/java/.../order/application/
│   │   ├── command/                       # 命令（写操作）
│   │   │   ├── CreateOrderCmd.java
│   │   │   └── CancelOrderCmd.java
│   │   ├── query/                         # 查询（读操作）
│   │   │   ├── OrderListQry.java
│   │   │   └── OrderDetailQry.java
│   │   ├── service/                       # 应用服务
│   │   │   └── OrderApplicationService.java
│   │   ├── event/                         # 事件处理
│   │   │   └── OrderEventListener.java
│   │   └── assembler/                     # 装配器
│   │       └── OrderAssembler.java
│   └── pom.xml
├── order-infrastructure/                  # 基础设施层
│   ├── src/main/java/.../order/infrastructure/
│   │   ├── persistence/                   # 持久化
│   │   │   ├── mapper/                    # MyBatis Mapper
│   │   │   │   └── OrderMapper.java
│   │   │   ├── po/                        # 持久化对象
│   │   │   │   └── OrderPO.java
│   │   │   └── repository/                # 仓储实现
│   │   │       └── OrderRepositoryImpl.java
│   │   ├── event/                         # 事件发布
│   │   │   └── SpringDomainEventPublisher.java
│   │   ├── mq/                            # 消息队列
│   │   │   └── OrderEventProducer.java
│   │   └── config/                        # 配置类
│   │       └── OrderConfig.java
│   └── pom.xml
└── order-interfaces/                      # 接口层
    ├── src/main/java/.../order/interfaces/
    │   ├── controller/                    # 控制器
    │   │   └── OrderController.java
    │   ├── dto/                           # 数据传输对象
    │   │   ├── CreateOrderRequest.java
    │   │   ├── OrderVO.java
    │   │   └── OrderItemDTO.java
    │   ├── assembler/                     # DTO装配器
    │   │   └── OrderDTOAssembler.java
    │   ├── exception/                     # 异常处理
    │   │   └── OrderExceptionHandler.java
    │   └── result/                        # 统一响应
    │       └── Result.java
    └── pom.xml

```

---

## 3. DDD分层架构详解

### 3.1 领域层（Domain Layer）

**核心原则**：领域层是整个系统的核心，不依赖任何外部框架。

#### 聚合根（Aggregate Root）
```java
/**
 * Order聚合根 - 订单领域核心
 */
@Entity
public class Order extends AggregateRoot<OrderId> {

    private OrderNo orderNo;           // 订单号（值对象）
    private OrderStatus status;        // 订单状态（值对象）
    private CustomerId customerId;     // 客户ID
    private Address shippingAddress;   // 收货地址（值对象）
    private Money totalAmount;         // 订单金额（值对象）
    private List<OrderItem> items;     // 订单明细

    // 工厂方法
    public static Order create(CreateOrderCmd cmd) {
        // 创建逻辑
    }

    // 领域行为
    public void cancel(String reason) {
        if (!canCancel()) {
            throw new OrderDomainException("订单不可取消");
        }
        this.status = OrderStatus.CANCELLED;
        this.registerEvent(new OrderCancelledEvent(this.id, reason));
    }

    // 领域规则
    private boolean canCancel() {
        return status == OrderStatus.PENDING
            || status == OrderStatus.CONFIRMED;
    }
}
```

#### 值对象（Value Object）
```java
/**
 * 地址值对象
 */
@ValueObject
public class Address {
    private String province;
    private String city;
    private String district;
    private String detail;

    // 工厂方法
    public static Address of(String province, String city, String district, String detail) {
        // 验证逻辑
        return new Address(province, city, district, detail);
    }
}
```

#### 仓储接口（Repository Interface）
```java
/**
 * 订单仓储接口 - 领域层定义，基础设施层实现
 */
public interface OrderRepository {

    Order findById(OrderId id);

    Order findByOrderNo(OrderNo orderNo);

    void save(Order order);

    void delete(OrderId id);

    Page<Order> findByCustomerId(CustomerId customerId, Pageable pageable);
}
```

### 3.2 应用层（Application Layer）

**核心原则**：应用层编排领域对象，协调业务用例，不包含业务逻辑。

```java
/**
 * 订单应用服务
 */
@Service
@Transactional
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final OrderAssembler orderAssembler;

    @Autowired
    public OrderApplicationService(OrderRepository orderRepository, OrderAssembler orderAssembler) {
        this.orderRepository = orderRepository;
        this.orderAssembler = orderAssembler;
    }

    /**
     * 创建订单
     */
    public OrderVO createOrder(CreateOrderRequest request) {
        // 1. 构建命令对象
        CreateOrderCmd cmd = orderAssembler.toCmd(request);

        // 2. 调用领域服务创建订单
        Order order = Order.create(cmd);

        // 3. 持久化
        orderRepository.save(order);

        // 4. 返回VO
        return orderAssembler.toVO(order);
    }

    /**
     * 取消订单
     */
    public OrderVO cancelOrder(String orderNo, String reason) {
        Order order = orderRepository.findByOrderNo(OrderNo.of(orderNo));
        order.cancel(reason);
        orderRepository.save(order);
        return orderAssembler.toVO(order);
    }
}
```

### 3.3 基础设施层（Infrastructure Layer）

**核心原则**：实现领域层定义的接口，提供技术能力。

```java
/**
 * 订单仓储实现
 */
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final DomainEventPublisher eventPublisher;

    @Override
    public void save(Order order) {
        // 1. 转换为PO
        OrderPO orderPO = OrderMapper.INSTANCE.toPO(order);

        // 2. 判断新增或更新
        if (order.isNew()) {
            orderMapper.insert(orderPO);
        } else {
            orderMapper.updateById(orderPO);
        }

        // 3. 同步保存订单项
        order.getItems().forEach(item -> {
            OrderItemPO itemPO = OrderItemMapper.INSTANCE.toPO(item);
            if (item.isNew()) {
                orderItemMapper.insert(itemPO);
            } else {
                orderItemMapper.updateById(itemPO);
            }
        });

        // 4. 发布领域事件
        order.getDomainEvents().forEach(event -> eventPublisher.publish(event));
    }
}
```

### 3.4 接口层（Interfaces Layer）

**核心原则**：处理外部请求，参数校验，格式转换。

```java
/**
 * 订单控制器
 */
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderApplicationService orderApplicationService;

    @PostMapping
    public Result<OrderVO> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        OrderVO orderVO = orderApplicationService.createOrder(request);
        return Result.success(orderVO);
    }

    @PostMapping("/{orderNo}/cancel")
    public Result<OrderVO> cancelOrder(
            @PathVariable String orderNo,
            @Valid @RequestBody CancelOrderRequest request) {
        OrderVO orderVO = orderApplicationService.cancelOrder(orderNo, request.getReason());
        return Result.success(orderVO);
    }

    @GetMapping("/{orderNo}")
    public Result<OrderVO> getOrder(@PathVariable String orderNo) {
        OrderVO orderVO = orderApplicationService.getOrderDetail(orderNo);
        return Result.success(orderVO);
    }
}
```

---

## 4. BFF层设计

### 4.1 BFF职责划分

| BFF类型 | 职责 | 数据策略 |
|--------|------|---------|
| Web BFF | 完整数据响应、复杂查询、大屏聚合 | 实时查询、深度聚合 |
| Mobile BFF | 精简数据响应、离线缓存、弱网适配 | 轻量数据、分页加载 |
| Admin BFF | 完整业务数据、批量操作、报表导出 | 完整数据、批量接口 |

### 4.2 BFF项目结构

```
gh-transport-gateway/
├── gateway-common/                    # 公共依赖
├── gateway-web/                       # Web BFF
│   ├── src/main/java/.../gateway/web/
│   │   ├── controller/                # Web端控制器
│   │   │   ├── OrderWebController.java
│   │   │   └── DashboardController.java
│   │   ├── feign/                     # Feign客户端调用领域服务
│   │   │   ├── OrderFeignClient.java
│   │   │   └── DispatchFeignClient.java
│   │   ├── dto/                       # Web专用DTO
│   │   │   ├── DashboardVO.java
│   │   │   └── WebOrderVO.java
│   │   └── config/                    # BFF配置
│   │       └── WebGatewayConfig.java
│   └── resources/
│       └── application-web.yml
├── gateway-mobile/                    # Mobile BFF
├── gateway-admin/                     # Admin BFF
└── pom.xml
```

### 4.3 Feign客户端定义

```java
/**
 * 订单服务Feign客户端（面向订单限界上下文）
 */
@FeignClient(name = "gh-transport-order")
public interface OrderFeignClient {

    @GetMapping("/api/v1/orders/{orderNo}")
    Result<OrderVO> getOrder(@PathVariable("orderNo") String orderNo);

    @PostMapping("/api/v1/orders")
    Result<OrderVO> createOrder(@RequestBody CreateOrderRequest request);

    @PostMapping("/api/v1/orders/{orderNo}/cancel")
    Result<OrderVO> cancelOrder(
            @PathVariable("orderNo") String orderNo,
            @RequestBody CancelOrderRequest request);
}

/**
 * 调度服务Feign客户端
 */
@FeignClient(name = "gh-transport-dispatch")
public interface DispatchFeignClient {
    // 调度相关接口
}
```

---

## 5. 微服务治理设计

> 微服务架构核心：服务注册发现、配置中心、限流熔断、认证授权、分布式事务

### 5.1 服务注册与发现（Nacos）

#### 服务启动注册
```yaml
# application.yml - 每个微服务都需要
spring:
  application:
    name: gh-transport-order  # 服务名，对应Nacos中的服务名

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  # Nacos地址
        namespace: dev               # 命名空间
        group: DEFAULT_GROUP         # 分组
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
        file-extension: yaml

# 暴露健康检查端点
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

#### Nacos 服务列表
| 服务名 | 命名空间 | 分组 | 描述 |
|--------|---------|------|------|
| gh-transport-gateway-web | dev | DEFAULT_GROUP | Web BFF |
| gh-transport-gateway-mobile | dev | DEFAULT_GROUP | Mobile BFF |
| gh-transport-gateway-admin | dev | DEFAULT_GROUP | Admin BFF |
| gh-transport-order | dev | DEFAULT_GROUP | 订单服务 |
| gh-transport-dispatch | dev | DEFAULT_GROUP | 调度服务 |
| gh-transport-transport | dev | DEFAULT_GROUP | 运输服务 |
| gh-transport-customer | dev | DEFAULT_GROUP | 客户管理 |
| gh-transport-inventory | dev | DEFAULT_GROUP | 库存管理 |
| gh-transport-auth | dev | DEFAULT_GROUP | 认证服务 |

### 5.2 配置中心（Nacos）

#### 配置管理规范
```
# Nacos 配置命名规范
Data ID: {service-name}.{env}.{file-extension}
例如：gh-transport-order.dev.yaml
```

#### 配置分组
| Data ID | 分组 | 说明 |
|---------|------|------|
| gh-transport-order.dev.yaml | DEFAULT_GROUP | 应用配置 |
| gh-transport-order-dev.yaml | DEFAULT_GROUP | 共享配置 |
| common-dev.yaml | DEFAULT_GROUP | 公共配置 |

#### 典型配置
```yaml
# gh-transport-order.dev.yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/gh_transport_order
    username: postgres
    password: ${ORDER_DB_PASSWORD:postgres}

  data:
    mongodb:
      uri: mongodb://localhost:27017/gh_transport

  redis:
    host: localhost
    port: 6379

# 应用配置
app:
  version: 1.0.0
  environment: dev

# 数据库配置
database:
  order:
    pool:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 30000
      max-lifetime: 1800000
```

### 5.3 负载均衡

#### Spring Cloud LoadBalancer 配置
```yaml
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false  # 禁用Ribbon，使用Spring Cloud LoadBalancer

# 服务实例选择策略
# 轮询、随机、加权、基于响应时间等
```

```java
/**
 * 自定义负载均衡策略
 */
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

### 5.4 限流熔断（Sentinel）

#### Sentinel 配置
```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8858  # Sentinel Dashboard
        port: 8719
      datasource:
        ds1:
          nacos:
            server-addr: localhost:8848
            dataId: gh-transport-sentinel-rules
            groupId: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```

#### 流控规则配置
```json
// Nacos: gh-transport-sentinel-rules.json
[
  {
    "resource": "createOrder",
    "limitApp": "default",
    "grade": 1,
    "count": 100,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  },
  {
    "resource": "getOrderDetail",
    "limitApp": "default",
    "grade": 1,
    "count": 200,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  }
]
```

#### 熔断降级
```java
/**
 * 熔断降级处理
 */
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{orderNo}")
    @SentinelResource(
        value = "getOrderDetail",
        blockHandler = "handleBlock",
        fallback = "handleFallback"
    )
    public Result<OrderVO> getOrder(@PathVariable String orderNo) {
        // 业务逻辑
    }

    /**
     * 流控/熔断降级处理
     */
    public Result<OrderVO> handleBlock(String orderNo, BlockException ex) {
        return Result.fail("FLOW_LIMIT", "请求过于频繁，请稍后重试");
    }

    /**
     * 业务异常降级
     */
    public Result<OrderVO> handleFallback(String orderNo, Throwable ex) {
        return Result.fail("SYSTEM_ERROR", "服务繁忙，请稍后重试");
    }
}
```

### 5.5 Feign 客户端配置

#### 完整 Feign 配置
```java
/**
 * Feign 配置
 */
@Configuration
public class FeignConfig {

    /**
     * 开启 Feign 熔断器
     */
    @Bean
    public Retryer retryer() {
        return new Retryer.Default(100, 1000, 3);
    }

    /**
     * 配置请求超时时间
     */
    @Bean
    public Request.Options options() {
        return new Request.Options(5000, 10000);
    }

    /**
     * 配置解码器
     */
    @Bean
    public Decoder decoder(ObjectProvider<HttpMessageConverters> messageConverters) {
        return new SpringDecoder(messageConverters);
    }

    /**
     * 配置编码器
     */
    @Bean
    public Encoder encoder(ObjectProvider<HttpMessageConverters> messageConverters) {
        return new SpringEncoder(messageConverters);
    }
}

/**
 * Feign 客户端 - 订单服务
 */
@FeignClient(
    name = "gh-transport-order",
    path = "/api/v1/orders",
    configuration = FeignConfig.class,
    fallbackFactory = OrderFeignClientFallbackFactory.class
)
public interface OrderFeignClient {

    @GetMapping("/{orderNo}")
    Result<OrderVO> getOrder(@PathVariable("orderNo") String orderNo);

    @PostMapping
    Result<OrderVO> createOrder(@RequestBody CreateOrderRequest request);

    @PostMapping("/{orderNo}/cancel")
    Result<OrderVO> cancelOrder(
        @PathVariable("orderNo") String orderNo,
        @RequestBody CancelOrderRequest request);
}

/**
 * Feign 降级工厂
 */
@Component
public class OrderFeignClientFallbackFactory implements FallbackFactory<OrderFeignClient> {

    @Override
    public OrderFeignClient create(Throwable cause) {
        return new OrderFeignClient() {
            @Override
            public Result<OrderVO> getOrder(String orderNo) {
                return Result.fail("SERVICE_UNAVAILABLE", "订单服务暂时不可用");
            }

            @Override
            public Result<OrderVO> createOrder(CreateOrderRequest request) {
                return Result.fail("SERVICE_UNAVAILABLE", "订单服务暂时不可用");
            }

            @Override
            public Result<OrderVO> cancelOrder(String orderNo, CancelOrderRequest request) {
                return Result.fail("SERVICE_UNAVAILABLE", "订单服务暂时不可用");
            }
        };
    }
}
```

### 5.6 统一认证授权

#### JWT 配置
```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key-here}
  expiration: 86400000  # 24小时
  refresh-expiration: 604800000  # 7天
  issuer: gh-transport
```

#### JWT 工具类
```java
/**
 * JWT 工具类
 */
@Component
public class JwtUtils {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private long expiration;

    /**
     * 生成 Token
     */
    public String generateToken(UserPrincipal principal) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .subject(principal.getUserId())
            .claim("username", principal.getUsername())
            .claim("roles", principal.getRoles())
            .claim("tenantId", principal.getTenantId())
            .issuedAt(now)
            .expiration(expiryDate)
            .signWith(SignatureAlgorithm.HS256, secret)
            .compact();
    }

    /**
     * 解析 Token
     */
    public Claims parseToken(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    }

    /**
     * 验证 Token
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}

/**
 * 认证过滤器
 */
@Component
public class AuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private AuthService authService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String token = extractToken(request);

        if (StringUtils.hasText(token) && jwtUtils.validateToken(token)) {
            Claims claims = jwtUtils.parseToken(token);
            UserPrincipal principal = authService.getPrincipal(claims);
            SecurityContextHolder.getContext().setAuthentication(principal);
        }

        chain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

#### 网关认证配置
```java
/**
 * 网关认证配置
 */
@Configuration
@EnableWebFlux
public class GatewayAuthConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder, AuthHandler authHandler) {
        return builder.routes()
            .route("auth-service", r -> r.path("/api/v1/auth/**")
                .uri("lb://gh-transport-auth"))
            .route("order-service", r -> r.path("/api/v1/orders/**")
                .filters(f -> f.filter(authHandler.validateToken()))
                .uri("lb://gh-transport-order"))
            // ... 其他路由
            .build();
    }
}
```

### 5.7 分布式事务（Seata）

#### Seata 配置
```yaml
# application.yml
spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: gh_transport_tx_group

seata:
  enabled: true
  application-id: gh-transport-order
  config:
    type: nacos
    nacos:
      server-addr: localhost:8848
      group: DEFAULT_GROUP
      data-id: seata-config.yaml
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      group: DEFAULT_GROUP
```

#### Saga 模式事务编排
```java
/**
 * Saga 模式 - 订单创建事务
 */
@Service
public class OrderSagaService {

    @Autowired
    private SagaContext sagaContext;

    /**
     * 创建订单 - Saga 编排
     */
    @GlobalTransactional
    public OrderVO createOrder(CreateOrderCmd cmd) {
        // 1. 创建订单
        Order order = Order.create(cmd);
        orderRepository.save(order);

        // 2. 扣减库存（Saga 调用）
        sagaContext.asyncCall(
            "inventory-service",
            "deductInventory",
            cmd.getItems(),
            compensation -> {
                // 补偿逻辑：库存返还
                inventoryService.revertInventory(cmd.getItems());
            }
        );

        // 3. 创建调度单
        Dispatch dispatch = dispatchService.createDispatch(order);

        return orderAssembler.toVO(order);
    }
}

/**
 * 库存服务 - Saga 接口
 */
@RestController
@RequestMapping("/api/v1/inventory")
public class InventoryController {

    @PostMapping("/deduct")
    public Result<Void> deductInventory(@Valid @RequestBody DeductInventoryRequest request) {
        // 扣减库存
        inventoryService.deduct(request.getItems());
        return Result.success();
    }

    @PostMapping("/revert")
    public Result<Void> revertInventory(@Valid @RequestBody RevertInventoryRequest request) {
        // 库存返还（补偿操作）
        inventoryService.revert(request.getItems());
        return Result.success();
    }
}
```

### 5.8 服务健康检查

#### 健康检查端点
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      group:
        readiness:
          show-details: always
        liveness:
          show-details: never
```

#### 自定义健康检查
```java
/**
 * 自定义健康检查
 */
@Component
public class CustomHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private MongoTemplate mongoTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Override
    public Health health() {
        try {
            // 检查数据库连接
            dataSource.getConnection().valid(1000);

            // 检查Redis连接
            redisTemplate.getConnectionFactory().getConnection().ping();

            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

### 5.9 微服务间通信模式

#### 同步调用 vs 异步事件
| 场景 | 调用方式 | 说明 |
|-----|---------|------|
| 用户注册后发送短信 | 异步事件 | 不阻塞主流程 |
| 创建订单扣减库存 | 异步事件（推荐）或 Saga | 最终一致性 |
| 获取用户信息 | 同步 Feign | 实时性要求高 |
| 订单状态更新通知 | 异步事件 | 不阻塞主流程 |

#### 调用链路追踪
```java
/**
 * 链路追踪 - OpenTelemetry/SkyWalking
 */
@Component
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(TracerProvider.noop())
            .build();
    }
}

/**
 * Feign 拦截器 - 传递 TraceID
 */
@Component
public class TracingFeignInterceptor implements RequestInterceptor {

    @Autowired
    private Tracer tracer;

    @Override
    public void apply(RequestTemplate template) {
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            template.header("X-Trace-Id", currentSpan.getSpanContext().getTraceId());
            template.header("X-Span-Id", currentSpan.getSpanContext().getSpanId());
        }
    }
}
```

---

## 6. 事件驱动设计

### 6.1 领域事件

```java
/**
 * 领域事件基类
 */
public abstract class DomainEvent {
    private final String eventId;
    private final LocalDateTime occurredOn;
    private final String aggregateId;

    protected DomainEvent(String aggregateId) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredOn = LocalDateTime.now();
        this.aggregateId = aggregateId;
    }
}

/**
 * 订单创建事件
 */
public class OrderCreatedEvent extends DomainEvent {
    private final String orderNo;
    private final String customerId;
    private final Money amount;
    private final List<String> itemIds;

    public OrderCreatedEvent(String orderNo, String customerId,
                           Money amount, List<String> itemIds) {
        super(orderNo);
        this.orderNo = orderNo;
        this.customerId = customerId;
        this.amount = amount;
        this.itemIds = itemIds;
    }
    // getters...
}
```

### 6.2 事件发布与消费

```java
/**
 * 领域事件发布者接口
 */
public interface DomainEventPublisher {
    void publish(DomainEvent event);
}

/**
 * 事件监听器示例
 */
@Component
public class OrderEventListener {

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private TransportService transportService;

    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 通知库存服务扣减库存
        inventoryService.deductInventory(event.getItemIds());

        // 通知运输服务安排配送
        transportService.arrangeDelivery(event.getOrderNo());
    }
}
```

---

## 7. 前端项目结构

### 7.1 Web端（Vue 3）

```
gh-transport-web/
├── src/
│   ├── api/                           # API调用层
│   │   ├── order.js                   # 订单API
│   │   ├── dispatch.js                # 调度API
│   │   └── index.js                   # API入口
│   ├── views/                         # 页面组件
│   │   ├── order/                     # 订单模块
│   │   │   ├── OrderList.vue
│   │   │   ├── OrderDetail.vue
│   │   │   └── CreateOrder.vue
│   │   └── dashboard/                 # 大屏模块
│   │       └── Dashboard.vue
│   ├── components/                    # 公共组件
│   ├── stores/                        # Pinia状态管理
│   ├── utils/                         # 工具函数
│   └── router/                        # 路由配置
├── package.json
└── vite.config.ts
```

### 7.2 Admin端（React）

```
gh-transport-admin/
├── src/
│   ├── api/                           # API调用层
│   │   ├── order.ts
│   │   └── index.ts
│   ├── pages/                         # 页面组件
│   │   ├── OrderManagement/
│   │   │   ├── OrderList.tsx
│   │   │   └── OrderDetail.tsx
│   │   └── SystemManagement/
│   ├── components/                    # 公共组件
│   ├── stores/                        # 状态管理
│   ├── hooks/                         # 自定义Hooks
│   └── router/                        # 路由配置
├── package.json
└── vite.config.ts
```

---

## 8. 数据库设计

### 8.1 PostgreSQL（结构化数据）

**订单表 (t_order)**
| 字段 | 类型 | 说明 |
|-----|------|------|
| id | UUID | 主键 |
| order_no | VARCHAR(32) | 订单号 |
| customer_id | UUID | 客户ID |
| status | VARCHAR(20) | 订单状态 |
| total_amount | DECIMAL(12,2) | 订单金额 |
| shipping_address | JSONB | 收货地址 |
| created_at | TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | 更新时间 |

**调度表 (t_dispatch)**
| 字段 | 类型 | 说明 |
|-----|------|------|
| id | UUID | 主键 |
| order_id | UUID | 关联订单ID |
| vehicle_id | UUID | 车辆ID |
| driver_id | UUID | 司机ID |
| status | VARCHAR(20) | 调度状态 |
| planned_time | TIMESTAMP | 计划时间 |
| actual_time | TIMESTAMP | 实际时间 |

### 7.2 MongoDB（文档数据）

**运输轨迹集合 (transport_track)**
```json
{
  "_id": ObjectId,
  "orderId": "order-uuid",
  "vehicleId": "vehicle-uuid",
  "location": {
    "lng": 116.404,
    "lat": 39.915
  },
  "speed": 60,
  "direction": 90,
  "timestamp": ISODate,
  "createdAt": ISODate
}
```

### 7.3 Redis（缓存）

| Key | 类型 | 说明 |
|-----|------|------|
| sso:session:{token} | Hash | 用户会话 |
| order:{orderNo} | Hash | 订单缓存 |
| inventory:{skuId} | String | 库存缓存 |

---

## 9. 微服务部署架构

### 9.1 整体部署架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              负载均衡 (Nginx)                            │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │  Web BFF     │  │ Mobile BFF   │  │ Admin BFF    │
            │  (8080)      │  │ (8081)       │  │ (8082)       │
            └──────────────┘  └──────────────┘  └──────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │  Auth Svc    │  │  Order Svc   │  │ Dispatch Svc │
            │  (8083)      │  │  (8084)      │  │ (8085)       │
            └──────────────┘  └──────────────┘  └──────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │ Transport    │  │ Customer     │  │ Inventory    │
            │ (8086)       │  │ (8087)       │  │ (8088)       │
            └──────────────┘  └──────────────┘  └──────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │   Nacos      │  │   Kafka      │  │  Sentinel    │
            │ (注册/配置)   │  │  (消息队列)   │  │  (限流)      │
            └──────────────┘  └──────────────┘  └──────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │  PostgreSQL  │  │   MongoDB    │  │    Redis     │
            │  (结构化)     │  │  (文档)      │  │   (缓存)      │
            └──────────────┘  └──────────────┘  └──────────────┘
```

### 9.2 服务注册状态监控

```bash
# 查看Nacos中注册的所有服务
curl -X GET "http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=100" \
  -H "Authorization: Bearer token"

# 查看某个服务的实例列表
curl -X GET "http://localhost:8848/nacos/v1/ns/instance/list?serviceName=gh-transport-order" \
  -H "Authorization: Bearer token"

# 查看Sentinel流控规则
curl -X GET "http://localhost:8858/flow/rules?app=gh-transport-order"

# 查看服务健康状态
curl -X GET "http://localhost:8084/actuator/health"
```

### 9.3 服务扩容与缩容

```bash
# 扩容订单服务（增加实例）
docker run -d \
  --name gh-transport-order-2 \
  -p 8084:8084 \
  -e NACOS_SERVER_ADDR=localhost:8848 \
  gh-transport-order:latest

# 查看服务实例
curl -X GET "http://localhost:8848/nacos/v1/ns/instance/list?serviceName=gh-transport-order"

# 自动扩缩容（Kubernetes HPA）
# apiVersion: autoscaling/v2
# kind: HorizontalPodAutoscaler
# metadata:
#   name: gh-transport-order-hpa
# spec:
#   scaleTargetRef:
#     apiVersion: apps/v1
#     kind: Deployment
#     name: gh-transport-order
#   minReplicas: 2
#   maxReplicas: 10
#   metrics:
#     - type: Resource
#       resource:
#         name: cpu
#         target:
#           type: Utilization
#           averageUtilization: 70
```

### 9.4 日志聚合（ELK）

```yaml
# docker-compose-elk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

  logstash:
    image: docker.elastic.co/logstash/logstash:8.10.0
    container_name: logstash
    ports:
      - "5000:5000"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline

volumes:
  es_data:
```

```yaml
# logstash/pipeline/logstash.conf
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  if [service] == "gh-transport-order" {
    mutate {
      add_field => { "index" => "gh-transport-order-%{+YYYY.MM.dd}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{index}"
  }
}
```

### 9.5 监控告警（Prometheus + Grafana）

```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'nacos'
    static_configs:
      - targets: ['localhost:8848']

  - job_name: 'gh-transport-order'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8084']

  - job_name: 'gh-transport-dispatch'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8085']

  - job_name: 'sentinel'
    static_configs:
      - targets: ['localhost:8858']
```

---

## 10. 开发流程指南

### 11.1 新增限界上下文步骤

1. **创建模块目录结构**
2. **定义领域模型**（Entity、ValueObject、Aggregate）
3. **定义仓储接口**（Repository Interface）
4. **实现应用服务**（Application Service）
5. **实现基础设施**（RepositoryImpl、Mapper）
6. **实现接口层**（Controller、DTO）
7. **配置BFF层**（FeignClient）
8. **编写单元测试**

### 11.2 新增功能步骤

1. **领域层**：定义聚合根、值对象、领域事件
2. **应用层**：编写Command/Query、ApplicationService
3. **基础设施层**：实现Repository、发布事件
4. **接口层**：Controller、DTO、参数校验
5. **BFF层**：FeignClient、聚合接口
6. **前端**：API调用、页面组件

---

## 12. 项目启动

### 12.1 后端启动

```bash
# 1. 启动基础设施
docker-compose up -d postgres redis mongodb kafka nacos

# 2. 编译项目
cd gh-transport
mvn clean install -DskipTests

# 3. 启动各个服务
mvn -pl gh-transport-gateway/gateway-web spring-boot:run
mvn -pl gh-transport-order/order-application spring-boot:run
# ... 依次启动其他服务
```

### 12.2 前端启动

```bash
# Web端
cd gh-transport-front/gh-transport-web
npm install
npm run dev

# Admin端
cd gh-transport-front/gh-transport-admin
npm install
npm run dev
```

---

## 13. 事务与幂等性设计

### 13.1 事务边界策略

**核心原则**：事务边界控制在应用层，领域层不管理事务。

#### 事务配置
```java
/**
 * 订单应用服务 - 事务边界
 */
@Service
@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    /**
     * 创建订单 - 事务方法
     * 如果 save() 或事件发布失败，整个事务回滚
     */
    public OrderVO createOrder(CreateOrderCmd cmd) {
        // 1. 调用领域服务创建订单
        Order order = Order.create(cmd);

        // 2. 持久化 - 在同一个事务中
        orderRepository.save(order);

        // 3. 事件会在事务提交后自动发布
        // 如果事务回滚，事件不会发布（Spring TransactionEventListener）
        return orderAssembler.toVO(order);
    }

    /**
     * 取消订单 - 事务方法
     */
    public OrderVO cancelOrder(CancelOrderCmd cmd) {
        Order order = orderRepository.findById(OrderId.of(cmd.getOrderId()));
        order.cancel(cmd.getReason());
        orderRepository.save(order);
        return orderAssembler.toVO(order);
    }
}
```

#### 事务传播行为

| 行为 | 说明 | 适用场景 |
|-----|------|---------|
| REQUIRED | 默认，有事务则加入，无则创建新事务 | 大多数操作 |
| REQUIRES_NEW | 总是创建新事务，独立提交/回滚 | 日志记录、审计 |
| NESTED | 嵌套事务，独立回滚 | 批量操作中的部分失败 |
| NOT_SUPPORTED | 无事务 | 只读查询 |

#### 跨限界上下文事务

**关键原则**：BC 之间**不要用分布式事务**，用**最终一致性**。

```java
// 场景：创建订单 + 扣减库存 + 安排调度
// 使用 Saga 模式或事件驱动

// 方案1：事件驱动（推荐）
@Service
@Transactional
public class OrderApplicationService {

    public OrderVO createOrder(CreateOrderCmd cmd) {
        Order order = Order.create(cmd);
        orderRepository.save(order);
        // 事件在事务提交后异步发送
        // 库存服务、调度服务独立处理
        return orderAssembler.toVO(order);
    }
}

// 方案2：本地事务表（开发阶段）
@Table(name = "outbox_event")
public class OutboxEvent {
    String id;
    String aggregateType;
    String aggregateId;
    String eventType;
    String payload;
    LocalDateTime createdAt;
    String status;  // PENDING / SENT / FAILED
}
```

### 10.2 幂等性设计

#### 幂等性场景

| 场景 | 风险 | 解决方案 |
|-----|------|---------|
| 用户连续点击提交按钮 | 重复创建订单 | 接口防重 + 订单号去重 |
| 网关重试请求 | 重复调用接口 | 请求ID去重 |
| 消息重发 | 重复消费 | 消息ID去重 |
| 分页游标重复 | 数据重复返回 | 游标去重 |

### 13.2 幂等性设计

#### 幂等性实现

```java
/**
 * 幂等性键 - Redis 实现
 */
@Component
public class IdempotentKey {

    private static final String IDEMPOTENT_PREFIX = "idempotent:";

    /**
     * 生成幂等性键
     * 格式：idempotent:{userId}:{bizType}:{bizKey}
     */
    public String generate(String userId, String bizType, String bizKey) {
        return IDEMPOTENT_PREFIX + userId + ":" + bizType + ":" + bizKey;
    }
}

/**
 * 幂等性注解
 */
@Aspect
@Component
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Idempotent {
    String bizType();
    int expireSeconds() default 300;
}

/**
 * 幂等性切面
 */
@Aspect
@Component
public class IdempotentAspect {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Around("@annotation(idempotent)")
    public Object around(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        // 1. 生成幂等性键
        String userId = SecurityContext.getCurrentUser().getId();
        String bizKey = generateBizKey(joinPoint.getArgs());
        String key = idempotentKey.generate(userId, idempotent.bizType(), bizKey);

        // 2. 检查是否已存在
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", idempotent.expireSeconds(), TimeUnit.SECONDS);

        if (Boolean.TRUE.equals(acquired)) {
            // 获取锁成功，执行方法
            try {
                return joinPoint.proceed();
            } finally {
                // 业务处理完成后删除（或者延长过期时间）
                // redisTemplate.delete(key);
            }
        } else {
            // 获取锁失败，返回已处理
            throw new IdempotentException("请求正在处理中，请勿重复提交");
        }
    }
}

/**
 * 订单控制器 - 幂等性示例
 */
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @PostMapping
    @Idempotent(bizType = "createOrder", expireSeconds = 300)
    public Result<OrderVO> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        // 幂等性键基于 request 中的唯一标识
        String bizKey = DigestUtils.md5Hex(
            request.getCustomerId() + request.getItemHash()
        );
        // ...
    }
}

/**
 * 订单号生成器 - 保证订单号唯一
 */
@Component
public class OrderNoGenerator {

    /**
     * 生成格式：ORD + 日期 + 流水号
     * 例如：ORD20250129000001
     */
    public String generate() {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String sequence = generateSequence();
        return "ORD" + date + StringUtils.leftPad(sequence, 8, '0');
    }

    private synchronized String generateSequence() {
        // Redis incr + 每日重置
        Long seq = redisTemplate.opsForValue().increment("order:seq:daily");
        return seq.toString();
    }
}
```

### 13.3 异常处理规范

#### 异常分类

| 异常类型 | 所在层 | 处理方式 |
|---------|-------|---------|
| DomainException | 领域层 | 业务规则校验失败 |
| ApplicationException | 应用层 | 应用级错误 |
| InfrastructureException | 基础设施层 | 技术错误（数据库、MQ等） |
| ValidationException | 接口层 | 参数校验失败 |

#### 异常处理

```java
/**
 * 领域异常 - 业务规则违反
 */
public class DomainException extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> context;

    public DomainException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }

    public DomainException(String errorCode, String message, Map<String, Object> context) {
        super(message);
        this.errorCode = errorCode;
        this.context = context;
    }
}

/**
 * 应用异常 - 应用级错误
 */
public class ApplicationException extends RuntimeException {
    private final String errorCode;
    private final ErrorLevel level;

    public enum ErrorLevel {
        WARN,   // 需要关注的错误
        ERROR,  // 系统错误
        FATAL   // 严重错误
    }
}

/**
 * 全局异常处理器
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 领域异常 - 返回业务错误码
     */
    @ExceptionHandler(DomainException.class)
    public Result<Void> handleDomainException(DomainException e) {
        log.warn("业务异常: {} - {}", e.getErrorCode(), e.getMessage());
        return Result.fail(e.getErrorCode(), e.getMessage());
    }

    /**
     * 参数校验异常
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining(", "));
        return Result.fail("VALIDATION_ERROR", message);
    }

    /**
     * 基础设施异常
     */
    @ExceptionHandler(InfrastructureException.class)
    public Result<Void> handleInfrastructureException(InfrastructureException e) {
        log.error("基础设施异常", e);
        return Result.fail("SYSTEM_ERROR", "系统繁忙，请稍后重试");
    }

    /**
     * 其他异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail("SYSTEM_ERROR", "系统错误，请联系管理员");
    }
}

/**
 * 统一响应结果
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Result<T> {
    private boolean success;
    private String code;
    private String message;
    private T data;
    private LocalDateTime timestamp;

    public static <T> Result<T> success(T data) {
        return new Result<>(true, "SUCCESS", null, data, LocalDateTime.now());
    }

    public static <T> Result<T> fail(String code, String message) {
        return new Result<>(false, code, message, null, LocalDateTime.now());
    }
}
```

### 13.4 ID 生成策略

#### ID 类型

| ID类型 | 生成方式 | 适用场景 |
|-------|---------|---------|
| 聚合根ID | UUID v7 / 雪花算法 | 内部主键 |
| 业务编号 | 业务前缀 + 日期 + 流水号 | 订单号、运单号等 |
| 外键ID | UUID | 关联查询 |

#### ID 生成实现

```java
/**
 * ID生成器
 */
@Component
public class IdGenerator {

    /**
     * 生成 UUID v7（带时间戳的UUID）
     * 优点：可以按时间排序，支持分库分表
     */
    public String generateUUIDv7() {
        // 方式1：使用库
        // return UUID.randomUUID().toString();

        // 方式2：雪花算法（推荐）
        return SnowflakeIdGenerator.nextId();
    }

    /**
     * 生成业务编号
     * 格式：前缀 + 日期 + 流水号
     */
    public String generateBizCode(String prefix) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        long seq = redisTemplate.opsForValue().increment("seq:" + prefix + ":" + date);
        return prefix + date + StringUtils.leftPad(String.valueOf(seq), 8, '0');
    }
}

/**
 * 雪花算法实现
 */
@Component
public class SnowflakeIdGenerator {

    private static final long WORKER_ID_BITS = 5L;
    private static final long DATACENTER_ID_BITS = 5L;
    private static final long SEQUENCE_BITS = 12L;

    private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;
    private static final long DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;
    private static final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS;

    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    @Value("${snowflake.worker-id:1}")
    public void setWorkerId(long workerId) {
        this.workerId = workerId;
    }

    @Value("${snowflake.datacenter-id:1}")
    public void setDatacenterId(long datacenterId) {
        this.datacenterId = datacenterId;
    }

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();

        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨，拒绝生成ID");
        }

        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & ((1 << SEQUENCE_BITS) - 1);
            if (sequence == 0) {
                // 同一毫秒内序列号耗尽，等待下一毫秒
                while (timestamp <= lastTimestamp) {
                    timestamp = System.currentTimeMillis();
                }
            }
        } else {
            sequence = 0L;
        }

        lastTimestamp = timestamp;

        return (timestamp << TIMESTAMP_LEFT_SHIFT)
            | (datacenterId << DATACENTER_ID_SHIFT)
            | (workerId << WORKER_ID_SHIFT)
            | sequence;
    }
}

/**
 * 领域ID - 值对象封装
 */
@ValueObject
public class OrderId {
    private final String value;

    public static OrderId of(String value) {
        return new OrderId(value);
    }

    public static OrderId generate() {
        return new OrderId(IdGenerator.generateUUIDv7());
    }
}

/**
 * 订单号 - 值对象
 */
@ValueObject
public class OrderNo {
    private final String value;

    private static final OrderNoGenerator GENERATOR = new OrderNoGenerator();

    public static OrderNo generate() {
        return new OrderNo(GENERATOR.generateBizCode("ORD"));
    }

    public static OrderNo of(String value) {
        return new OrderNo(value);
    }
}
```

---

## 11. 开发阶段简化方案

### 11.1 单体模式启动

对于第一次实践 DDD，建议**先做单体应用**，熟悉分层后再拆分微服务。

#### 目录结构调整

```
gh-transport/
├── gh-transport-parent/
├── gh-transport-common/
├── gh-transport-gateway/           # 网关（可选）
├── gh-transport-server/            # 单体服务（包含所有BC）
│   ├── server-domain/              # 领域层
│   │   ├── order/
│   │   ├── dispatch/
│   │   └── ...
│   ├── server-application/         # 应用层
│   ├── server-infrastructure/      # 基础设施层
│   ├── server-interfaces/          # 接口层
│   └── src/main/resources/
│       └── application.yml
└── gh-transport-front/             # 前端项目
```

#### 配置文件

```yaml
# application.yml
spring:
  application:
    name: gh-transport-server

  # PostgreSQL 配置
  datasource:
    url: jdbc:postgresql://localhost:5432/gh_transport
    username: postgres
    password: postgres

  # MongoDB 配置
  data:
    mongodb:
      uri: mongodb://localhost:27017/gh_transport

  # Redis 配置
  data:
    redis:
      host: localhost
      port: 6379

# Nacos 配置（可选）
nacos:
  server-addr: localhost:8848
  config:
    namespace: dev

# 应用配置
server:
  port: 8080

# 日志配置
logging:
  level:
    com.ghtransport: DEBUG
    org.springframework: INFO
```

### 11.2 本地事件表（替代 Kafka）

开发阶段用本地事件表 + 定时任务，简化部署。

```sql
-- 事件表
CREATE TABLE domain_event (
    id VARCHAR(64) PRIMARY KEY,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    event_payload TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP,
    retry_count INT DEFAULT 0
);

-- 事件索引
CREATE INDEX idx_domain_event_status ON domain_event(status);
CREATE INDEX idx_domain_event_aggregate ON domain_event(aggregate_type, aggregate_id);
```

```java
/**
 * 本地事件发布者
 */
@Component
public class LocalEventPublisher implements DomainEventPublisher {

    @Autowired
    private DomainEventRepository eventRepository;

    @Override
    public void publish(DomainEvent event) {
        DomainEventEntity entity = new DomainEventEntity();
        entity.setId(UUID.randomUUID().toString());
        entity.setAggregateType(event.getClass().getSimpleName());
        entity.setAggregateId(event.getAggregateId());
        entity.setEventType(event.getClass().getName());
        entity.setEventPayload(JSON.toJSONString(event));
        entity.setStatus("PENDING");
        eventRepository.save(entity);
    }
}

/**
 * 事件处理定时任务
 */
@Component
public class EventProcessor {

    @Autowired
    private DomainEventRepository eventRepository;

    @Autowired
    private ApplicationContext applicationContext;

    @Scheduled(fixedRate = 5000)
    public void processEvents() {
        List<DomainEventEntity> events = eventRepository.findByStatus("PENDING");

        for (DomainEventEntity entity : events) {
            try {
                // 根据事件类型分发
                Object event = JSON.parseObject(entity.getEventPayload(), Class.forName(entity.getEventType()));
                Object handler = applicationContext.getBean(event.getClass());

                // 调用处理器
                if (handler instanceof OrderCreatedEventHandler) {
                    ((OrderCreatedEventHandler) handler).handle((OrderCreatedEvent) event);
                }

                entity.setStatus("PROCESSED");
                entity.setProcessedAt(LocalDateTime.now());
            } catch (Exception e) {
                entity.setRetryCount(entity.getRetryCount() + 1);
                if (entity.getRetryCount() > 3) {
                    entity.setStatus("FAILED");
                }
            }
            eventRepository.save(entity);
        }
    }
}
```

### 14.3 快速启动命令

```bash
# 1. 启动数据库（Docker）
docker run -d \
  --name gh-transport-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=gh_transport \
  -p 5432:5432 \
  postgres:15

docker run -d \
  --name gh-transport-mongo \
  -p 27017:27017 \
  mongo:6

docker run -d \
  --name gh-transport-redis \
  -p 6379:6379 \
  redis:7-alpine

# 2. 启动后端服务
cd gh-transport
mvn clean package -DskipTests

java -jar gh-transport-server/target/gh-transport-server.jar

# 3. 启动前端
cd gh-transport-front/gh-transport-web
npm install
npm run dev

# 4. 访问
# Web端: http://localhost:3000
# API文档: http://localhost:8080/swagger-ui.html
```

---

## 15. 开发规范

### 15.1 代码规范

```java
// 1. 聚合根命名规范
public class Order extends AggregateRoot<OrderId> { ... }
public class Dispatch extends AggregateRoot<DispatchId> { ... }

// 2. 值对象命名规范
@ValueObject
public class OrderNo { ... }        // 编号
@ValueObject
public class OrderStatus { ... }     // 状态
@ValueObject
public class Address { ... }        // 地址

// 3. 应用服务命名规范
public class OrderApplicationService { ... }
public class DispatchApplicationService { ... }

// 4. 仓储接口命名规范（领域层）
public interface OrderRepository { ... }

// 5. 仓储实现命名规范（基础设施层）
public class OrderRepositoryImpl implements OrderRepository { ... }
```

### 15.2 包结构规范

```
com.ghtransport.{bc}.domain           # 领域层
├── aggregate/                         # 聚合根
│   └── Order.java
├── entity/                            # 实体
├── valueobject/                       # 值对象
├── repository/                        # 仓储接口
├── service/                           # 领域服务
├── event/                             # 领域事件
└── exception/                         # 领域异常

com.ghtransport.{bc}.application      # 应用层
├── command/                           # 命令
├── query/                             # 查询
├── service/                           # 应用服务
├── event/                             # 事件处理
└── assembler/                         # 装配器

com.ghtransport.{bc}.infrastructure    # 基础设施层
├── persistence/                       # 持久化
│   ├── mapper/
│   ├── po/
│   └── repository/
├── mq/                                # 消息队列
└── config/                            # 配置

com.ghtransport.{bc}.interfaces       # 接口层

### 15.2 包结构规范
├── controller/                        # 控制器
├── dto/                               # DTO
├── assembler/                         # 装配器
├── exception/                         # 异常处理
└── result/                            # 统一响应
```

### 15.2 包结构规范

---

## 16. 验收标准

- [ ] 项目结构符合 DDD 四层架构
- [ ] 每个限界上下文独立微服务部署
- [ ] 领域层不依赖外部框架
- [ ] 应用层只编排业务用例，不包含业务逻辑
- [ ] BFF 层按端定制 API
- [ ] 领域事件异步解耦（Kafka）
- [ ] 前后端分离开发
- [ ] 单元测试覆盖核心逻辑
- [ ] 幂等性设计防止重复提交
- [ ] 异常处理统一规范
- [ ] ID 生成策略明确
- [ ] 服务注册到 Nacos
- [ ] 配置统一管理（Nacos）
- [ ] 限流熔断配置（Sentinel）
- [ ] 接口认证授权（JWT）
- [ ] 分布式事务 Saga 编排
- [ ] 服务健康检查正常
- [ ] 日志链路追踪

---

*文档版本: 2.0*
*创建日期: 2025-01-29*
*更新日期: 2025-01-29*
*备注: 更新为微服务架构版本*
