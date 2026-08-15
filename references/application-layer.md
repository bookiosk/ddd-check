# Application Layer Specification


## 目录
- [Part A: Base Rules (All Modes)](#part-a-base-rules-all-modes)
  - [A.1 Package Structure](#a1-package-structure)
  - [A.2 Core Positioning](#a2-core-positioning)
  - [A.3 CQRS (Command Query Responsibility Segregation)](#a3-cqrs-command-query-responsibility-segregation)
  - [A.4 Exception Boundary — APP as Unified Catch Point](#a4-exception-boundary-—-app-as-unified-catch-point)
  - [A.5 Parameters and Return Values](#a5-parameters-and-return-values)
  - [A.6 DTO Location](#a6-dto-location)
  - [A.7 Dependency Rules](#a7-dependency-rules)
  - [A.8 Behavior Constraints](#a8-behavior-constraints)
  - [A.9 Assembler](#a9-assembler)
- [Part B: Write Mode](#part-b-write-mode)
  - [B.1 Naming](#b1-naming)
  - [B.2 Call Chain](#b2-call-chain)
  - [B.3 Orchestration Steps](#b3-orchestration-steps)
  - [B.4 Template](#b4-template)
- [Part C: Read Mode](#part-c-read-mode)
  - [C.1 Naming](#c1-naming)
  - [C.2 Data Sources](#c2-data-sources)
  - [C.3 Template (Hybrid Query)](#c3-template-hybrid-query)
- [Part D: Pure Calculate Mode](#part-d-pure-calculate-mode)
  - [D.1 Core Principle](#d1-core-principle)
  - [D.2 Naming](#d2-naming)
  - [D.3 Call Chain](#d3-call-chain)
  - [D.4 Orchestration Steps](#d4-orchestration-steps)
  - [D.5 Template](#d5-template)
- [Part E: Rule+Calculate Mode](#part-e-rule+calculate-mode)
  - [E.1 Core Principle](#e1-core-principle)
  - [E.2 Naming](#e2-naming)
  - [E.3 Call Chain](#e3-call-chain)
  - [E.4 Template](#e4-template)

## Part A: Base Rules (All Modes)

### A.1 Package Structure

```
application/
└── {business}/               # Per business domain (order, booking, refund)
    ├── scenario/             # Scene orchestration (AppService implementations)
    └── assembler/            # DTO ↔ Domain object mapping
```

### A.2 Core Positioning

**Application layer is scene orchestration (unstable); Domain layer is core business logic (stable).**

- Application layer orchestrates domain services, Adaptors, and Repositories per business scene — this is where requirements change most frequently
- Domain layer (DomainService, aggregates) encapsulates **stable core business rules** — invariant across scenes
- Same domain service method reused by **multiple Application scene methods**; scene differences expressed in Application orchestration logic

**Example:** `orderDomainService.createOrder()` is the single stable domain method. Application provides two scene methods:
- `createOrderOnline` — validates inventory and price before creating (complex orchestration)
- `createOrderManual` — creates order directly (simple orchestration)

Future scenes (e.g., batch import) only need new Application methods — domain service stays unchanged.

### A.3 CQRS (Command Query Responsibility Segregation)

Application layer splits into:
- **Command (Write):** `{AggregateName}AppService extends ApplicationCmdService` — business changes only
- **Query (Read):** `{AggregateName}QueryAppService extends ApplicationQueryService` — data queries only

**Rule:** Command layer MUST NOT call Query layer interfaces.

### A.4 Exception Boundary — APP as Unified Catch Point

APP 层是异常处理的统一收口。Domain、outAdaptor 的阻断型异常以及 `requestDTO.check()` 的参数校验异常都在这里被捕获并转换为 `ResultDO`。

```
Domain / outAdaptor          APP (boundary)              Adaptor (inAdaptor)
      │                          │                            │
      │  throw BizException  →   │  catch → ResultDO.fail()  │
      │  throw AggregateEx   →   │  catch → ResultDO.fail()  │
      │  return ResultDO     →   │  分支处理/透传             │
      │                          │                            │
      │                          │  ← Response / ResultDO ──  │
```

**APP 方法结构规则：** 除日志操作（`log.xxx`）外，所有业务操作 MUST 放在 `try` 块内；`catch` 块只负责记日志 + 转换为 `ResultDO.fail()`。

**APP catches:**
| Source | Exception | APP Action |
|---|---|---|
| RequestDTO 参数自校验 | `IllegalArgumentException` | `ResultDO.fail("PARAM_ERROR", e.getMessage())` |
| DomainService | `BizException` | `ResultDO.fail(e.getCode(), e.getMsg())` |
| Aggregate/Entity | `AggregateException` | `ResultDO.fail(e.getCode(), e.getMsg())` |
| outAdaptor | Exception | Scene-specific: retry / fallback / `ResultDO.fail()` |
| Any | `Throwable` | Log + `ResultDO.fail("SYSTEM_ERROR", "System error")` |

**APP handles ResultDO (分支型):**
| Domain/Adaptor returns | APP Action |
|---|---|
| `ResultDO.fail("DUPLICATE", msg, existingOrder)` | 提取 `existingOrder` 返回给上游 |
| `ResultDO.fail("CACHE_MISS", key, null)` | 降级走其他数据源 |

**阻断型 vs 分支型 — 优先抛异常:**

| 失败是否终止整个 APP 方法 | 机制 |
|---|---|
| 失败必须终止整个 APP 方法（事件到此结束，无需数据回传） | **抛异常（阻断型）** — APP catch 后转 `ResultDO.fail()` |
| 失败但需特殊处理 / 走其他逻辑继续完成正常链路 | **返回 `ResultDO`（分支型）** — APP 判断后走其他链路 |

**决策原则：优先抛异常。** 只有当抛异常会导致逻辑不对（例如失败后仍需走降级 / 兜底 / 其他链路继续正常流程）时，才允许返回 `ResultDO`。

**APP 自身不抛异常给 Adaptor** — all returns are `ResultDO<T>`.

### A.5 Parameters and Return Values

| Rule | Spec |
|------|------|
| Input parameter | MUST use `RequestDTO` (a concrete `{MethodName}RequestDTO` subclass defined in `client` layer, e.g. `CreateOrderRequestDTO extends BaseDTO`), FORBIDDEN: primitive types or Map |
| Parameter validation | Via `requestDTO.check()` — a `void` method that throws `IllegalArgumentException` on failure. FORBIDDEN: writing validation logic in AppService |
| Return value | Always `ResultDO<T>`, generic is ResponseDTO (a concrete `{MethodName}ResponseDTO` subclass defined in `client` layer, e.g. `CreateOrderResponseDTO extends BaseDTO`) |
| Error handling | APP is the **exception boundary**: catches Domain/Adaptor blocking exceptions → `ResultDO.fail()`. Never throws to Adaptor. |

> **IMPORTANT — `RequestDTO`/`ResponseDTO` are naming suffixes, not concrete classes.** Never write a class literally named `RequestDTO` or `ResponseDTO`. The parameter is always a concrete subclass like `CreateOrderRequestDTO extends BaseDTO`; the return generic is always a concrete subclass like `CreateOrderResponseDTO extends BaseDTO`. Both are defined in the `client` module.

```java
// Parameter self-validation (by RequestDTO itself) — void, throws on failure
requestDTO.check();

// Success
return ResultDO.buildSuccessResult(responseDTO);

// Error
return ResultDO.buildFailResult("ORDER_NOT_FOUND", "Order not found");
```

### A.6 DTO Location

| Scenario | Location | Notes |
|----------|----------|-------|
| Exposed as external API | `client` module `req/` and `res/` packages | External callers depend on these DTOs |
| Application-internal only | top-level `model` module | Not exposed externally, only for internal orchestration |

**Rule:** If a DTO is transparently passed to external callers by Input Adaptor (Controller, HSF, etc.), put it in `client`. If only used within Application layer flow, put it in the **top-level `model` module** (the shared internal module), NOT in an `application/model/` sub-package.

### A.7 Dependency Rules

**Allowed:**
- `domain` module
- `client` module
- `model` module
- I/O-free utility classes

**Forbidden:**
- Other business domain module JARs
- Middleware (message middleware, OSS, etc.)

### A.8 Behavior Constraints

**Allowed:**
- Call DomainService
- Use Repository interface
- Call Adaptor for external data (cross-domain queries)
- DTO ↔ Domain object conversion (via Assembler)

**Forbidden:**
- Direct database access (use Repository)
- Direct external service access (use Adaptor)
- Core business logic (belongs in DomainService or aggregates)

**Important distinction:** Application layer MAY include **orchestration flow control** (e.g., checking Adaptor results, early returns with error codes). This is scene orchestration, not business logic. Core business logic = aggregate state transitions, business calculation formulas, rule matching — these MUST stay in Domain layer.

### A.9 Assembler

**Naming:** `{AggregateName}Assembler` or `{BusinessScene}Assembler`

**Usage in APP:** RequestDTO → Domain Param (before calling DomainService), and Aggregate → ResponseDTO (before returning).

```java
public class OrderAssembler {

    /** RequestDTO → Domain Param */
    public static CreateOrderParam toParam(CreateOrderRequestDTO requestDTO) {
        CreateOrderParam param = new CreateOrderParam();
        param.setProductId(requestDTO.getProductId());
        param.setQuantity(requestDTO.getQuantity());
        return param;
    }

    /** Aggregate → ResponseDTO */
    public static CreateOrderResponseDTO toResponseDTO(OrderAggregate order) {
        CreateOrderResponseDTO responseDTO = new CreateOrderResponseDTO();
        responseDTO.setOrderId(order.getId());
        responseDTO.setStatus(order.getStatus());
        return responseDTO;
    }
}
```

**Assembler call site inside AppService method (all inside try block):**

```java
try {
    requestDTO.check();

    // DTO → Param via Assembler, then call DomainService
    CreateOrderParam param = OrderAssembler.toParam(requestDTO);
    OrderAggregate order = orderDomainService.createOrder(param);

    // Aggregate → ResponseDTO via Assembler
    CreateOrderResponseDTO responseDTO = OrderAssembler.toResponseDTO(order);
    return ResultDO.buildSuccessResult(responseDTO);
} catch (IllegalArgumentException e) {
    log.error("Invalid param, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
} catch (BizException | AggregateException e) {
    log.error("Create order business error, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult(e.getCode(), e.getMsg());
} catch (Throwable e) {
    log.error("Create order system error, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
}
```

---

## Part B: Write Mode

### B.1 Naming

| Rule | Spec | Example |
|------|------|---------|
| Interface | `{AggregateName}AppService` | `OrderAppService` |
| Implementation | `{AggregateName}AppServiceImpl` | `OrderAppServiceImpl` |
| Inheritance | MUST extend `ApplicationCmdService` | — |
| Method naming | Business verbs, NOT technical verbs (`save`, `update`) | `createOrder`, `confirmPayment` |
| Request DTO | `{MethodName}RequestDTO` | `CreateOrderRequestDTO` |
| Response DTO | `{MethodName}ResponseDTO` | `CreateOrderResponseDTO` |

### B.2 Call Chain

**Simple (e.g., manual order):**
```
Input Adaptor → AppService → DomainService → Repository
```

**Complex (e.g., online order with external validation):**
```
Input Adaptor → AppService → Adaptor (validate inventory/price)
                           → DomainService → Repository
```

### B.3 Orchestration Steps

1. Parameter self-validation (`requestDTO.check()` — void, throws `IllegalArgumentException`)
2. Fetch external data or validate preconditions via Adaptor (per scene needs)
3. Convert DTO to Domain Param (via Assembler)
4. Call DomainService (may throw blocking exception — caught by APP)
5. Handle DomainService ResultDO for 分支型 scenarios (duplicate check, fallback)
6. Build ResponseDTO and return `ResultDO.success()`

**Exception handling in APP (all business ops inside try, only logging + conversion in catch):**
```java
try {
    // Parameter self-validation — throws IllegalArgumentException
    requestDTO.check();

    // Assembler: DTO → Param
    CreateOrderParam param = OrderAssembler.toParam(requestDTO);

    // Call DomainService / outAdaptor — throws on failure
    OrderAggregate order = orderDomainService.createOrder(param);

    // Assembler: Aggregate → ResponseDTO
    CreateOrderResponseDTO responseDTO = OrderAssembler.toResponseDTO(order);
    return ResultDO.buildSuccessResult(responseDTO);
} catch (IllegalArgumentException e) {
    log.error("Invalid param, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
} catch (BizException | AggregateException e) {
    log.error("Business error, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult(e.getCode(), e.getMsg());
} catch (Throwable e) {
    log.error("System error, requestDTO: {}", requestDTO, e);
    return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
}
```

### B.4 Template

See Part A.2 for complete online vs. manual order creation example.

---

## Part C: Read Mode

### C.1 Naming

| Rule | Spec | Example |
|------|------|---------|
| Interface | `{AggregateName}QueryAppService` | `OrderQueryAppService` |
| Implementation | `{AggregateName}QueryAppServiceImpl` | `OrderQueryAppServiceImpl` |
| Inheritance | MUST extend `ApplicationQueryService` | — |
| Method naming | Verbs with clear query intent, avoid abstract words (`process`) | `queryOrderList`, `getOrderDetail` |

### C.2 Data Sources

| Scenario | Data Source | Notes |
|----------|------------|-------|
| Intra-domain query | Repository | Get aggregate or Result, convert to DTO via Assembler |
| Cross-domain query | Adaptor | Get 3rd-party data, return directly or trim |
| Hybrid query | Repository + Adaptor | Query intra-domain first, supplement with Adaptor, assemble |

### C.3 Template (Hybrid Query)

```java
@Slf4j
@Service
public class OrderQueryAppServiceImpl implements OrderQueryAppService {

    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private LogisticsAdaptor logisticsAdaptor;

    @Override
    public ResultDO<GetOrderDetailResponseDTO> getOrderDetail(GetOrderDetailRequestDTO requestDTO) {
        try {
            // 1. Self-validate — void, throws IllegalArgumentException on failure
            requestDTO.check();

            // 2. Query intra-domain order data — throws on failure, returns simple aggregate
            OrderAggregate order = orderRepository.findById(requestDTO.getOrderId());

            // 3. Cross-domain logistics query via Adaptor — throws on failure, returns simple DTO
            LogisticsInfoResponseDTO logistics = logisticsAdaptor.queryLogistics(order.getLogisticsNo());

            // 4. Assemble response
            GetOrderDetailResponseDTO responseDTO = OrderAssembler.toGetOrderDetailResponseDTO(order, logistics);
            return ResultDO.buildSuccessResult(responseDTO);
        } catch (IllegalArgumentException e) {
            log.error("Invalid param, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
        } catch (BizException | AggregateException e) {
            log.error("Get order detail failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult(e.getCode(), e.getMsg());
        } catch (Throwable e) {
            log.error("Get order detail failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
        }
    }
}
```

> **Note:** Repository and Adaptor here both throw on failure (阻断型) because a failure terminates the whole APP method — no data is needed to continue. Only when the failure must be handled specially (fallback, branch to another data source) should they return `ResultDO` (分支型).

---

## Part D: Pure Calculate Mode

### D.1 Core Principle
**One calculation method = one class.** Each calculation method gets its own independent class with single responsibility.

### D.2 Naming

| Rule | Spec | Example |
|------|------|---------|
| Interface | `{Verb}QueryAppService` | `FeePreCalculateQueryAppService` |
| Implementation | `{Verb}QueryAppServiceImpl` | `FeePreCalculateQueryAppServiceImpl` |
| Inheritance | MUST extend `ApplicationQueryService` | — |
| Method naming | Verb | `feePreCalculate` |

### D.3 Call Chain
```
Input Adaptor → {Verb}QueryAppService → DomainService → calculate & return Result
```

### D.4 Orchestration Steps
1. Parameter self-validation (`requestDTO.check()` — void, throws `IllegalArgumentException`)
2. Fetch external data via Adaptor (if needed)
3. Assemble Param, call DomainService for computation
4. Convert Result to ResponseDTO and return

### D.5 Template
```java
@Slf4j
@Service
public class FeePreCalculateQueryAppServiceImpl implements FeePreCalculateQueryAppService {

    @Autowired
    private ExchangeRateAdaptor exchangeRateAdaptor;
    @Autowired
    private FeeCalculateDomainService feeCalculateDomainService;

    @Override
    public ResultDO<FeePreCalculateResponseDTO> feePreCalculate(FeePreCalculateRequestDTO requestDTO) {
        try {
            // 1. Self-validate — void, throws IllegalArgumentException
            requestDTO.check();

            // 2. Cross-domain exchange rate query — throws on failure, returns simple DTO
            ExchangeRateResponseDTO rate = exchangeRateAdaptor.queryExchangeRate(requestDTO.getCurrency());

            // 3. Assemble calculation params
            FeeCalculateParam param = FeePreCalculateAssembler.toParam(requestDTO, rate);

            // 4. Call DomainService — throws on failure, returns simple Result
            FeeCalculateResult result = feeCalculateDomainService.calculate(param);

            // 5. Convert to response
            return ResultDO.buildSuccessResult(FeePreCalculateAssembler.toResponseDTO(result));
        } catch (IllegalArgumentException e) {
            log.error("Invalid param, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
        } catch (BizException | AggregateException e) {
            log.error("Fee pre-calculation failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult(e.getCode(), e.getMsg());
        } catch (Throwable e) {
            log.error("Fee pre-calculation failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
        }
    }
}
```

---

## Part E: Rule+Calculate Mode

### E.1 Core Principle
Same as Pure Calculate: **one calculation method = one class.** Naming starts with **verb**, not aggregate name.

### E.2 Naming

| Rule | Spec | Example |
|------|------|---------|
| Interface | `{Verb}QueryAppService` | `SubsidyPreCalculateQueryAppService` |
| Implementation | `{Verb}QueryAppServiceImpl` | `SubsidyPreCalculateQueryAppServiceImpl` |
| Inheritance | MUST extend `ApplicationQueryService` | — |
| Method naming | Verb | `subsidyPreCalculate` |

### E.3 Call Chain
```
Input Adaptor → {Verb}QueryAppService → DomainService → Repository.queryRules()
                                                       → Aggregate.matchRule()/calculate() → return Result
```

### E.4 Template
```java
@Slf4j
@Service
public class SubsidyPreCalculateQueryAppServiceImpl implements SubsidyPreCalculateQueryAppService {

    @Autowired
    private UserLevelAdaptor userLevelAdaptor;
    @Autowired
    private SubsidyCalculateDomainService subsidyCalculateDomainService;

    @Override
    public ResultDO<SubsidyPreCalculateResponseDTO> subsidyPreCalculate(SubsidyPreCalculateRequestDTO requestDTO) {
        try {
            // 1. Self-validate — void, throws IllegalArgumentException
            requestDTO.check();

            // 2. Cross-domain user level query — throws on failure, returns simple DTO
            UserLevelResponseDTO level = userLevelAdaptor.queryUserLevel(requestDTO.getUserId());

            // 3. Assemble params
            SubsidyCalculateParam param = SubsidyPreCalculateAssembler.toParam(requestDTO, level);

            // 4. Call DomainService (match rules + calculate) — throws on failure, returns simple Result
            SubsidyCalculateResult result = subsidyCalculateDomainService.calculate(param);

            // 5. Convert to response
            return ResultDO.buildSuccessResult(SubsidyPreCalculateAssembler.toResponseDTO(result));
        } catch (IllegalArgumentException e) {
            log.error("Invalid param, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("PARAM_ERROR", e.getMessage());
        } catch (BizException | AggregateException e) {
            log.error("Subsidy pre-calculation failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult(e.getCode(), e.getMsg());
        } catch (Throwable e) {
            log.error("Subsidy pre-calculation failed, requestDTO: {}", requestDTO, e);
            return ResultDO.buildFailResult("SYSTEM_ERROR", "System error");
        }
    }
}
```
