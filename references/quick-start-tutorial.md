# Quick-Start Tutorial — End-to-End "Create Order"


## 目录
- [Scenario](#scenario)
- [File Manifest](#file-manifest)
- [Step 1: Client Layer — Request DTO](#step-1-client-layer-—-request-dto)
- [Step 2: Domain Layer — Aggregate, DomainService, Repository Interface](#step-2-domain-layer-—-aggregate-domainservice-repository-interface)
  - [Aggregate](#aggregate)
  - [Entity](#entity)
  - [Param](#param)
  - [DomainService Interface](#domainservice-interface)
  - [DomainService Implementation](#domainservice-implementation)
  - [Repository Interface](#repository-interface)
- [Step 3: Application Layer — Scene Orchestration](#step-3-application-layer-—-scene-orchestration)
  - [Adaptor Interfaces (defined in Application, per Application needs)](#adaptor-interfaces-defined-in-application-per-application-needs)
  - [AppService Interface](#appservice-interface)
  - [AppService Implementation](#appservice-implementation)
  - [Assembler](#assembler)
- [Step 4: Infrastructure Layer — Persistence](#step-4-infrastructure-layer-—-persistence)
  - [PO](#po)
  - [Mapper](#mapper)
  - [Converter](#converter)
  - [Repository Implementation](#repository-implementation)
- [Step 5: Adaptor Layer — Input & Output](#step-5-adaptor-layer-—-input-&-output)
  - [Input Adaptor — HTTP Entry Point](#input-adaptor-—-http-entry-point)
  - [Output Adaptor — External Service Calls](#output-adaptor-—-external-service-calls)
- [Step 6: Data Flow Summary](#step-6-data-flow-summary)
- [How to Add a New Scene](#how-to-add-a-new-scene)

Walk through building a complete "create online order" feature across all 6 layers. Copy-paste ready.

---

## Scenario

User places an online order. The system:
1. Receives HTTP POST request with order details
2. Validates input
3. Checks inventory (external service)
4. Verifies price (external service)
5. Applies business rules: minimum amount 100, non-VIP max 50000
6. Creates order and persists to database
7. Returns order ID and status

**Mode:** Write Mode
**Call chain:** `Controller → AppService → Adaptor (inventory/price) → DomainService → Aggregate → Repository → DB`

---

## File Manifest

```
# Layer: client
client/order/req/CreateOrderRequestDTO.java

# Layer: model (already exists, reference only)
model/common/DomesticIntlEnum.java

# Layer: domain
domain/order/model/aggregate/OrderAggregate.java
domain/order/model/entity/OrderItemEntity.java
domain/order/model/param/CreateOrderParam.java
domain/order/service/OrderDomainService.java
domain/order/service/OrderDomainServiceImpl.java
domain/order/repository/OrderRepository.java

# Layer: application
application/order/scenario/OrderAppService.java
application/order/scenario/OrderAppServiceImpl.java
application/order/assembler/OrderAssembler.java

# Layer: infrastructure
infrastructure/order/repository/OrderRepositoryImpl.java
infrastructure/order/mysql/po/OrderPO.java
infrastructure/order/mysql/po/OrderItemPO.java
infrastructure/order/mysql/mapper/OrderMapper.java
infrastructure/order/mysql/mapper/OrderItemMapper.java
infrastructure/order/converter/OrderConverter.java

# Layer: adaptor (output — called by Application)
application/order/adaptor/InventoryAdaptor.java          # interface in application
application/order/adaptor/PriceAdaptor.java               # interface in application
adaptor/order/output/InventoryAdaptorImpl.java            # impl in adaptor
adaptor/order/output/PriceAdaptorImpl.java                # impl in adaptor

# Layer: adaptor (input — entry point)
adaptor/order/input/OrderController.java
```

---

## Step 1: Client Layer — Request DTO

```java
// client/order/req/CreateOrderRequestDTO.java
package com.example.client.order.req;

@Data
@EqualsAndHashCode(callSuper = true)
public class CreateOrderRequestDTO extends BaseDTO {
    private static final long serialVersionUID = 1L;

    private Long buyerId;
    private String productId;
    private String productName;
    private Integer quantity;
    private Long price;
    private Long amount;
    private Boolean isVip;

    public void check() {
        if (productId == null || productId.isEmpty()) {
            throw new IllegalArgumentException("Product ID required");
        }
        if (quantity == null || quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be > 0");
        }
        if (price == null || price <= 0) {
            throw new IllegalArgumentException("Price must be > 0");
        }
        if (amount == null || amount <= 0) {
            throw new IllegalArgumentException("Amount must be > 0");
        }
    }
}
```

---

## Step 2: Domain Layer — Aggregate, DomainService, Repository Interface

### Aggregate

```java
// domain/order/model/aggregate/OrderAggregate.java
package com.example.domain.order.model.aggregate;

@Data
@EqualsAndHashCode(callSuper = true)
public class OrderAggregate extends BaseAggregate<Long> {
    private String orderNo;
    private Long buyerId;
    private String productId;
    private Integer quantity;
    private Long amount;
    private OrderStatusEnum status;  // PENDING, PAID, CANCELLED
    private Date gmtCreate;
    private List<OrderItemEntity> items;

    /** Create order — all business rules HERE, single source of truth */
    public void create(CreateOrderParam param) {
        if (param == null) {
            throw new AggregateException("Param must not be null");
        }
        if (param.getAmount() < 100) {
            throw new AggregateException("AMOUNT_TOO_LOW", "Minimum order amount is 100");
        }
        if (param.getAmount() > 50000 && !Boolean.TRUE.equals(param.getIsVip())) {
            throw new AggregateException("LIMIT_EXCEEDED", "Non-VIP max amount is 50000");
        }

        this.orderNo = "ORD" + System.currentTimeMillis();
        this.buyerId = param.getBuyerId();
        this.productId = param.getProductId();
        this.quantity = param.getQuantity();
        this.amount = param.getAmount();
        this.status = OrderStatusEnum.PENDING;
        this.gmtCreate = new Date();
    }

    /** Check if order can be cancelled */
    @JSONField(serialize = false)
    public boolean canCancel() {
        return this.status == OrderStatusEnum.PENDING;
    }

    /** Calculate total amount from items — aggregate delegates to entity methods */
    @JSONField(serialize = false)
    public Long calculateTotalAmount() {
        if (items == null || items.isEmpty()) {
            return this.amount;
        }
        return items.stream()
            .mapToLong(OrderItemEntity::calculateSubtotal)
            .sum();
    }
}
```

### Entity

```java
// domain/order/model/entity/OrderItemEntity.java
package com.example.domain.order.model.entity;

@Data
@EqualsAndHashCode(callSuper = true)
public class OrderItemEntity extends BaseEntity<Long> {
    private Field<Long> orderId;
    private Field<String> productName;
    private Field<Long> price;
    private Field<Integer> quantity;
    private Field<ItemStatusValue> status;

    /** Business method — payment confirmation, NOT a CRUD setter */
    public void confirmPayment() {
        if (this.status.get() == null || !this.status.get().canPay()) {
            throw new AggregateException("Item " + getId() + " cannot be paid");
        }
        this.status = Field.of(ItemStatusValue.PAID);
    }

    /** Business method — cancellation */
    public void cancel() {
        if (this.status.get() == null || !this.status.get().canCancel()) {
            throw new AggregateException("Item " + getId() + " cannot be cancelled");
        }
        this.status = Field.of(ItemStatusValue.CANCELLED);
    }

    /** Calculate subtotal (side-effect-free) */
    public Long calculateSubtotal() {
        return this.price.get() * this.quantity.get();
    }
}
```

### Param

```java
// domain/order/model/param/CreateOrderParam.java
package com.example.domain.order.model.param;

@Data
@EqualsAndHashCode(callSuper = true)
public class CreateOrderParam extends BaseParam {
    private Long buyerId;
    private String productId;
    private String productName;
    private Integer quantity;
    private Long price;
    private Long amount;
    private Boolean isVip;
}
```

### DomainService Interface

```java
// domain/order/service/OrderDomainService.java
package com.example.domain.order.service;

public interface OrderDomainService {
    OrderAggregate createOrder(CreateOrderParam param);
}
```

### DomainService Implementation

```java
// domain/order/service/OrderDomainServiceImpl.java
package com.example.domain.order.service;

@Slf4j
@Service
public class OrderDomainServiceImpl implements OrderDomainService {

    @Resource
    private OrderRepository orderRepository;

    @Override
    public OrderAggregate createOrder(CreateOrderParam param) {
        LevelLock lock = orderRepository.buildLock("order:create:" + param.getBuyerId());
        try {
            if (!lock.tryLock()) {
                throw new BizException("LOCK_FAIL", "Failed to acquire lock, retry");
            }

            // Create aggregate — business rules execute inside
            // AggregateException propagates to APP layer (DomainService does NOT catch)
            OrderAggregate order = new OrderAggregate();
            order.create(param);

            // Persist — throws on failure
            orderRepository.save(order);

            return order;
        } finally {
            lock.unlock();
        }
        // No catch — blocking exceptions propagate to APP layer
    }
}
```

### Repository Interface

```java
// domain/order/repository/OrderRepository.java
package com.example.domain.order.repository;

public interface OrderRepository extends AggregateRepository<OrderAggregate, Long> {
    void save(OrderAggregate aggregate);
    LevelLock buildLock(String lockKey);
    OrderAggregate query(OrderQuery query);
}
```

---

## Step 3: Application Layer — Scene Orchestration

### Adaptor Interfaces (defined in Application, per Application needs)

```java
// application/order/adaptor/InventoryAdaptor.java
package com.example.application.order.adaptor;

public interface InventoryAdaptor {
    InventoryCheckResponseDTO checkInventory(String productId, Integer quantity);
}
```

```java
// application/order/adaptor/PriceAdaptor.java
package com.example.application.order.adaptor;

public interface PriceAdaptor {
    PriceCheckResponseDTO checkPrice(String productId, Long expectedPrice);
}
```

### AppService Interface

```java
// application/order/scenario/OrderAppService.java
package com.example.application.order.scenario;

public interface OrderAppService extends ApplicationCmdService {
    ResultDO<CreateOrderResponseDTO> createOrderOnline(CreateOrderRequestDTO requestDTO);
}
```

### AppService Implementation

```java
// application/order/scenario/OrderAppServiceImpl.java
package com.example.application.order.scenario;

@Slf4j
@Service
public class OrderAppServiceImpl implements OrderAppService {

    @Autowired
    private OrderDomainService orderDomainService;
    @Autowired
    private InventoryAdaptor inventoryAdaptor;
    @Autowired
    private PriceAdaptor priceAdaptor;

    @Override
    public ResultDO<CreateOrderResponseDTO> createOrderOnline(CreateOrderRequestDTO req) {
        try {
            // 1. Self-validation (RequestDTO owns its validation) — void, throws on failure
            req.check();

            // 2. Scene-specific: check inventory (via Adaptor) — returns simple DTO, throws on failure
            InventoryCheckResponseDTO inv = inventoryAdaptor.checkInventory(req.getProductId(), req.getQuantity());
            if (!inv.isSufficient()) {
                return ResultDO.buildFailResult("INVENTORY_NOT_ENOUGH", "Insufficient inventory");
            }

            // 3. Scene-specific: verify price (via Adaptor) — returns simple DTO, throws on failure
            PriceCheckResponseDTO price = priceAdaptor.checkPrice(req.getProductId(), req.getPrice());
            if (!price.isValid()) {
                return ResultDO.buildFailResult("PRICE_INVALID", "Price changed, please refresh");
            }

            // 4. Call domain service — business rules are in aggregate, not here — returns simple aggregate, throws on failure
            CreateOrderParam param = OrderAssembler.toParam(req);
            OrderAggregate order = orderDomainService.createOrder(param);

            // 5. Return response
            return ResultDO.buildSuccessResult(OrderAssembler.toCreateOrderResponseDTO(order));
        } catch (IllegalArgumentException e) {
            log.error("Invalid param, req: {}", req, e);
            return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
        } catch (BizException | AggregateException e) {
            log.error("Create order business error, req: {}", req, e);
            return ResultDO.buildFailResult(e.getCode(), e.getMsg());
        } catch (Exception e) {
            log.error("Create order online error, req: {}", req, e);
            return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
        }
    }
}
```

### Assembler

```java
// application/order/assembler/OrderAssembler.java
package com.example.application.order.assembler;

public class OrderAssembler {

    public static CreateOrderParam toParam(CreateOrderRequestDTO req) {
        CreateOrderParam param = new CreateOrderParam();
        param.setBuyerId(req.getBuyerId());
        param.setProductId(req.getProductId());
        param.setProductName(req.getProductName());
        param.setQuantity(req.getQuantity());
        param.setPrice(req.getPrice());
        param.setAmount(req.getAmount());
        param.setIsVip(req.getIsVip());
        return param;
    }

    public static CreateOrderResponseDTO toCreateOrderResponseDTO(OrderAggregate aggregate) {
        CreateOrderResponseDTO dto = new CreateOrderResponseDTO();
        dto.setOrderId(aggregate.getId());
        dto.setOrderNo(aggregate.getOrderNo());
        dto.setStatus(aggregate.getStatus().name());
        dto.setAmount(aggregate.getAmount());
        return dto;
    }
}
```

---

## Step 4: Infrastructure Layer — Persistence

### PO

```java
// infrastructure/order/mysql/po/OrderPO.java
package com.example.infrastructure.order.mysql.po;

@Data
public class OrderPO {
    private Long id;
    private String orderNo;
    private Long buyerId;
    private String productId;
    private Integer quantity;
    private Long amount;
    private Integer orderStatus;  // DB value: 0=PENDING, 1=PAID, 2=CANCELLED
    private Date gmtCreate;
    private Date gmtModified;
}
```

### Mapper

```java
// infrastructure/order/mysql/mapper/OrderMapper.java
package com.example.infrastructure.order.mysql.mapper;

public interface OrderMapper {
    int insert(OrderPO po);
    int updateById(OrderPO po);
    OrderPO selectById(Long id);
}
```

### Converter

```java
// infrastructure/order/converter/OrderConverter.java
package com.example.infrastructure.order.converter;

public class OrderConverter {

    /** PO → Aggregate — pure field mapping + enum conversion */
    public static OrderAggregate toAggregate(OrderPO po) {
        if (po == null) return null;
        OrderAggregate aggregate = new OrderAggregate();
        aggregate.setId(po.getId());  // ID backfill — framework persistence access (BaseAggregate.setId is protected)
        aggregate.setOrderNo(po.getOrderNo());
        aggregate.setBuyerId(po.getBuyerId());
        aggregate.setProductId(po.getProductId());
        aggregate.setQuantity(po.getQuantity());
        aggregate.setAmount(po.getAmount());
        aggregate.setStatus(OrderStatusEnum.getByCode(po.getOrderStatus()));
        aggregate.setGmtCreate(po.getGmtCreate());
        return aggregate;
    }

    /** Aggregate → PO — pure field mapping + enum conversion */
    public static OrderPO toPO(OrderAggregate aggregate) {
        if (aggregate == null) return null;
        OrderPO po = new OrderPO();
        po.setId(aggregate.getId());
        po.setOrderNo(aggregate.getOrderNo());
        po.setBuyerId(aggregate.getBuyerId());
        po.setProductId(aggregate.getProductId());
        po.setQuantity(aggregate.getQuantity());
        po.setAmount(aggregate.getAmount());
        if (aggregate.getStatus() != null) {
            po.setOrderStatus(aggregate.getStatus().getCode());
        }
        po.setGmtCreate(aggregate.getGmtCreate());
        po.setGmtModified(new Date());
        return po;
    }
}
```

### Repository Implementation

```java
// infrastructure/order/repository/OrderRepositoryImpl.java
package com.example.infrastructure.order.repository;

@Slf4j
@Component
public class OrderRepositoryImpl implements OrderRepository {

    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private OrderItemMapper orderItemMapper;

    @Override
    public void save(OrderAggregate aggregate) {
        try {
            OrderPO po = OrderConverter.toPO(aggregate);

            if (aggregate.getId() == null) {
                orderMapper.insert(po);
                aggregate.setId(po.getId());  // backfill generated ID
            } else {
                int affected = orderMapper.updateById(po);
                if (affected == 0) {
                    throw new BizException("UPDATE_FAIL", "Update failed, data may be stale");
                }
            }

            // Save child entities
            if (CollectionUtils.isNotEmpty(aggregate.getItems())) {
                for (OrderItemEntity item : aggregate.getItems()) {
                    OrderItemPO itemPO = OrderItemConverter.toPO(item, aggregate.getId());
                    if (item.getId() == null) {
                        orderItemMapper.insert(itemPO);
                        item.setId(itemPO.getId());
                    } else {
                        orderItemMapper.updateById(itemPO);
                    }
                }
            }
        } catch (BizException e) {
            throw e;
        } catch (Exception e) {
            log.error("Save order failed, aggregateId: {}", aggregate.getId(), e);
            throw new BizException("DB_SAVE_ERROR", "Save order error");
        }
    }

    @Override
    public LevelLock buildLock(String lockKey) {
        return new LevelLock(lockKey);
    }

    @Override
    public OrderAggregate query(OrderQuery query) {
        try {
            OrderPO po = orderMapper.selectById(query.getId());
            if (po == null) {
                throw new BizException("ORDER_NOT_FOUND", "Order not found");
            }
            OrderAggregate aggregate = OrderConverter.toAggregate(po);
            // Load child entities
            List<OrderItemPO> itemPOs = orderItemMapper.selectByOrderId(po.getId());
            aggregate.setItems(OrderItemConverter.toEntityList(itemPOs));
            return aggregate;
        } catch (BizException e) {
            throw e;
        } catch (Exception e) {
            log.error("Query order failed, query: {}", query, e);
            throw new BizException("DB_QUERY_ERROR", "Query order error");
        }
    }
}
```

---

## Step 5: Adaptor Layer — Input & Output

### Input Adaptor — HTTP Entry Point

```java
// adaptor/order/input/OrderController.java
package com.example.adaptor.order.input;

@RestController
@RequestMapping("/api/order")
public class OrderController {

    @Autowired
    private OrderAppService orderAppService;

    @PostMapping("/create")
    public ResultDO<CreateOrderResponseDTO> createOrder(@RequestBody CreateOrderRequestDTO req) {
        return orderAppService.createOrderOnline(req);
    }
}
```

### Output Adaptor — External Service Calls

```java
// adaptor/order/output/InventoryAdaptorImpl.java
package com.example.adaptor.order.output;

@Component
public class InventoryAdaptorImpl implements InventoryAdaptor {

    @Autowired
    private ThirdPartyInventoryClient inventoryClient;

    @Override
    public InventoryCheckResponseDTO checkInventory(String productId, Integer quantity) {
        ThirdPartyInventoryResponse response = inventoryClient.check(productId, quantity);
        // Anti-corruption: only return fields Application needs
        InventoryCheckResponseDTO dto = new InventoryCheckResponseDTO();
        dto.setSufficient(response.getRemainingQty() >= quantity);
        dto.setRemainingQuantity(response.getRemainingQty());
        return dto;
    }
}
```

```java
// adaptor/order/output/PriceAdaptorImpl.java
package com.example.adaptor.order.output;

@Component
public class PriceAdaptorImpl implements PriceAdaptor {

    @Autowired
    private ThirdPartyPriceClient priceClient;

    @Override
    public PriceCheckResponseDTO checkPrice(String productId, Long expectedPrice) {
        ThirdPartyPriceResponse response = priceClient.getCurrentPrice(productId);
        PriceCheckResponseDTO dto = new PriceCheckResponseDTO();
        dto.setValid(Objects.equals(expectedPrice, response.getCurrentPrice()));
        dto.setCurrentPrice(response.getCurrentPrice());
        return dto;
    }
}
```

---

## Step 6: Data Flow Summary

```
HTTP POST /api/order/create
  │
  ▼
OrderController
  │  CreateOrderRequestDTO
  ▼
OrderAppServiceImpl.createOrderOnline()
  │  1. req.check()                        ← self-validation
  │  2. inventoryAdaptor.checkInventory()   ← external (Adaptor)
  │  3. priceAdaptor.checkPrice()           ← external (Adaptor)
  │  4. OrderAssembler.toParam()            ← DTO → Param
  ▼
OrderDomainServiceImpl.createOrder()
  │  1. buildLock + tryLock                 ← concurrency
  │  2. new OrderAggregate().create(param)  ← business rules
  │  3. orderRepository.save(aggregate)     ← persist
  ▼
OrderRepositoryImpl.save()
  │  1. OrderConverter.toPO()               ← Aggregate → PO
  │  2. orderMapper.insert(po)              ← DB insert
  │  3. Backfill ID
  ▼
  ResultDO<OrderAggregate>
  │
  ▼
OrderAssembler.toCreateOrderResponseDTO()   ← Aggregate → ResponseDTO
  │
  ▼
  ResultDO<CreateOrderResponseDTO> → HTTP 200 JSON
```

> **Note:** In the flow above, `req.check()` is void and throws on failure; `inventoryAdaptor.checkInventory()` / `priceAdaptor.checkPrice()` / `orderDomainService.createOrder()` / `orderRepository.save()` all return simple types and throw on failure, which APP's try/catch catches and converts to `ResultDO`.

---

## How to Add a New Scene

**Scenario:** "Manual order entry" — no inventory or price checks.

Add ONE method to `OrderAppServiceImpl`:

```java
@Override
public ResultDO<CreateOrderResponseDTO> createOrderManual(CreateOrderManualRequestDTO req) {
    try {
        req.check();

        CreateOrderParam param = OrderAssembler.toParam(req);
        OrderAggregate order = orderDomainService.createOrder(param);

        return ResultDO.buildSuccessResult(OrderAssembler.toCreateOrderResponseDTO(order));
    } catch (IllegalArgumentException e) {
        log.error("Invalid param, req: {}", req, e);
        return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
    } catch (BizException | AggregateException e) {
        log.error("Create order business error, req: {}", req, e);
        return ResultDO.buildFailResult(e.getCode(), e.getMsg());
    } catch (Exception e) {
        log.error("Create order manual error, req: {}", req, e);
        return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
    }
}
```

No changes to domain layer. No changes to infrastructure. No duplicated business rules. Just scene orchestration differences in Application layer.
