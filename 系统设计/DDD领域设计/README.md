# DDD 领域驱动设计 README

> 这是一份可落地的 DDD 实战指南与项目 README 模板。适用于中大型业务系统（交易、营销、订单、结算等），帮助团队统一语言、划分边界、搭建分层、制定一致性策略与编码规范。

- 读者对象：后端开发、架构师、产品、测试
- 目标：从需求到模型，从模型到代码，形成可演进的高内聚系统

------

## 目录

- 为什么选择 DDD
- 术语速览（战略设计与战术设计）
- 分层架构与依赖约束
- 限界上下文与上下文映射（Context Map）
- 聚合建模与领域对象规范
- 领域事件、CQRS 与事件驱动
- 一致性与事务（ACID、Saga、Outbox、幂等）
- 目录结构与编码规范
- 示例代码（以“订单”聚合为例）
- 从需求到代码：团队协作流程
- 测试策略
- 可观测性与运维
- 版本化与演进
- 常见反模式与检查清单
- 贡献指南与 ADR 模板

------

## 为什么选择 DDD（Domain-Driven Design）🎯

- 聚焦核心领域：把有限的工程资源投入在最能体现差异化竞争力的业务能力上。
- 分而治之：通过限界上下文隔离复杂性，降低耦合与回归风险。
- 统一语言：业务与技术对齐，减少需求理解偏差。
- 易于演进：模型驱动架构，支持灰度、扩展与重构。

------

## 术语速览 🧭

- 战略设计（Strategic Design）
  - 限界上下文 Bounded Context：模型的边界，边界内语言一致、规则统一。
  - 上下文映射 Context Map：上下文之间的关系，如上游/下游、ACL、防腐、共享内核等。
  - 统一语言 Ubiquitous Language：业务与代码共享的术语体系。
- 战术设计（Tactical Design）
  - 聚合 Aggregate：强一致事务边界，保证不变量的对象集合（唯一聚合根）。
  - 实体 Entity：有标识的业务对象（生命周期、状态变化）。
  - 值对象 Value Object：无标识、不可变，用于表达值与概念。
  - 领域服务 Domain Service：跨实体/聚合的纯领域规则。
  - 仓储 Repository：聚合的持久化抽象。
  - 领域事件 Domain Event：领域内发生的事实，通常不可变、可溯源。
  - 应用服务 Application Service：用例编排、事务边界、调用协调，不包含领域规则。
  - 防腐层 ACL：对接外部上下文的翻译层，防止模型污染。
  - CQRS：命令与查询分离，写走领域模型，读走投影/缓存。

------

## 分层架构与依赖约束 🧱

- 接口层（API/Adapter/BFF）：DTO、参数校验、认证授权、协议适配
- 应用层（Application）：用例编排、事务、发命令与接事件、调用仓储与领域服务
- 领域层（Domain）：实体、值对象、聚合根、领域服务、领域事件（不依赖基础设施）
- 基础设施层（Infrastructure）：仓储实现、消息、数据库、外部系统的 ACL

依赖方向（自上而下）：
Interface -> Application -> Domain <- Infrastructure（通过注入/接口倒置）

------

## 限界上下文与上下文映射

示例上下文（可按业务替换）：

- 订单（Order）：下单、支付状态、取消、售后发起
- 促销定价（Pricing）：规则、组合、叠加、预算
- 库存（Inventory）：台账、预占、扣减、返还
- 支付（Payment）：渠道聚合、对账、回调
- 优惠券（Coupon）：发券、领券、锁券/核销
- 清结算（Settlement）：分账与对账

上下文关系（Context Map）常见模式：

- Customer-Supplier（顾客-供应）：Order 下游依赖 Pricing 的报价能力
- Conformist（顺从者）：弱势上下文被动适配上游模型
- Anticorruption Layer（防腐层）：引入翻译层对接外部支付网关
- Shared Kernel（共享内核）：极小可共享模型（如 Money、Currency）
- Open Host Service（开放主机服务）：向外提供稳定契约（API/事件）

------

## 聚合建模与领域对象规范

聚合设计要点：

- 单一职责：聚合根对外暴露行为与状态，不变量在聚合内原子维护。
- 小而精：避免“巨无霸聚合”。跨聚合引用使用 ID，而非对象引用。
- 事务边界：同一事务只修改一个聚合；跨聚合用事件/Saga 协调。
- 不变量示例：订单应付 = 行合计 + 运费 - 优惠；状态机不可越权跳转。

值对象规范：

- 不可变（immutable）、可比较（equals/hashCode）、自带校验（如金额非负、币种一致）
- 示例：Money、Address、TimeWindow、Percentage、QuoteSignature

领域事件规范：

- Past-tense 命名：OrderPaid、CouponLocked
- 携带最小足够上下文（ids、关键数值、时间）
- 可序列化、可重放；事件版本化支持演进

------

## 领域事件、CQRS 与事件驱动

- 领域事件（Domain Event）：上下文内的事实，驱动本上下文内的副作用或改变状态。
- 集成事件（Integration Event）：跨上下文传播的公共事件（语义稳定、版本化）。
- Outbox Pattern：写业务数据与事件至同一事务表，异步可靠投递 MQ（Kafka/Pulsar）。
- CQRS：
  - 写模型：严格遵守聚合边界和不变量。
  - 读模型：为了查询性能构建的投影（Redis/ES/ClickHouse），最终一致。

------

## 一致性与事务

- 聚合内：本地 ACID 事务，乐观锁/版本号控制并发。
- 跨聚合/跨上下文：
  - Saga 编排（Orchestration）或舞蹈（Choreography）
  - TCC：Try-Confirm-Cancel 适合库存预占、券锁定等资源场景
  - Outbox + 幂等消费：确保至少一次投递与有界重试
- 幂等策略：
  - 命令幂等键（idempotencyKey），支付回调按 tradeNo 去重
  - 事件消费者用“业务唯一键 + 事件类型”去重

------

## 目录结构与编码规范

推荐代码结构（每个限界上下文独立）：

- contexts/
  - order/
    - interface/ # REST/gRPC 适配层，DTO
    - application/ # 用例、事务、Saga 协调器
    - domain/ # 实体、VO、聚合、领域服务、事件
    - infrastructure/ # 仓储实现、消息、DB、ACL
    - bootstrap/ # 依赖注入、装配与启动
  - pricing/
  - inventory/
- shared-kernel/ # Money、Result、Page、Errors
- platform/ # 消息、配置、日志、追踪、Outbox

命名与约束：

- 领域对象禁止依赖基础设施（仅依赖 shared-kernel）
- 值对象用不可变类并封装校验
- 聚合以领域行为命名方法：order.pay(), coupon.lock()，避免 setXxx 暴露状态
- DTO 与领域对象隔离，转换放在 application/adapter

------

## 示例代码（以“订单”聚合为例）

值对象 Money：

```Java
public final class Money {
  private final BigDecimal amount;
  private final String currency;

  public Money(BigDecimal amount, String currency) {
    if (amount == null || amount.scale() > 2 || amount.compareTo(BigDecimal.ZERO) < 0) {
      throw new IllegalArgumentException("invalid amount");
    }
    this.amount = amount.stripTrailingZeros();
    this.currency = Objects.requireNonNull(currency);
  }

  public Money add(Money other) { ensureCurrency(other); return new Money(amount.add(other.amount), currency); }
  public Money sub(Money other) { ensureCurrency(other); return new Money(amount.subtract(other.amount), currency); }
  public boolean gte(Money other) { ensureCurrency(other); return amount.compareTo(other.amount) >= 0; }

  private void ensureCurrency(Money other) {
    if (!currency.equals(other.currency)) throw new IllegalArgumentException("currency mismatch");
  }
  // equals/hashCode/toString ...
}
```

领域事件：

```Java
public interface DomainEvent {
  Instant occurredAt();
  String version();
}

public record OrderPaid(UUID orderId, UUID buyerId, Money payAmount, Instant at) implements DomainEvent {
  public Instant occurredAt() { return at; }
  public String version() { return "v1"; }
}
```

订单聚合：

```Java
public class Order {
  public enum Status { PENDING_PAYMENT, PAID, CANCELED }

  private final UUID id;
  private final UUID buyerId;
  private Status status;
  private final List<OrderLine> lines = new ArrayList<>();
  private Money itemsTotal;
  private Money shippingFee;
  private Money discountTotal;
  private Money payAmount;
  private final List<DomainEvent> events = new ArrayList<>();

  private Order(UUID id, UUID buyerId) { this.id = id; this.buyerId = buyerId; }

  public static Order place(UUID buyerId, List<OrderLine> lines, Money shipping, Money discount) {
    Order o = new Order(UUID.randomUUID(), buyerId);
    o.lines.addAll(lines);
    o.itemsTotal = lines.stream().map(OrderLine::lineTotal).reduce(new Money(BigDecimal.ZERO, "CNY"), Money::add);
    o.shippingFee = shipping;
    o.discountTotal = discount;
    o.payAmount = o.itemsTotal.add(o.shippingFee).sub(o.discountTotal);
    o.assertInvariants();
    o.status = Status.PENDING_PAYMENT;
    return o;
  }

  public void markPaid() {
    if (status != Status.PENDING_PAYMENT) throw new IllegalStateException("illegal status");
    this.status = Status.PAID;
    this.events.add(new OrderPaid(id, buyerId, payAmount, Instant.now()));
  }

  private void assertInvariants() {
    if (payAmount.gte(new Money(new BigDecimal("0"), "CNY")) == false) {
      throw new IllegalStateException("payAmount < 0");
    }
  }

  public List<DomainEvent> pullEvents() { var copy = List.copyOf(events); events.clear(); return copy; }
  // getters...
}

public record OrderLine(UUID skuId, int qty, Money unitPrice) {
  public Money lineTotal() { return new Money(unitPrice.amount().multiply(BigDecimal.valueOf(qty)), unitPrice.currency()); }
}
```

仓储与应用服务：

```Java
public interface OrderRepository {
  Optional<Order> findById(UUID id);
  void save(Order order); // 包含新增与乐观锁更新
}

public class PlaceOrderHandler {
  private final OrderRepository orderRepo;
  private final OutboxPublisher outbox;
  private final PricingClient pricingAcl;

  @Transactional
  public UUID handle(PlaceOrder cmd) {
    var quote = pricingAcl.verify(cmd.quoteId(), cmd.signature()); // ACL 防腐层
    var order = Order.place(cmd.buyerId(), cmd.toLines(quote), quote.shipping(), quote.discount());
    orderRepo.save(order);
    outbox.publish(order.pullEvents()); // Outbox 事务消息
    return order.getId();
  }
}
```

Outbox（简化伪代码）：

```Java
@Transactional
void saveAndPublish(Aggregate agg) {
  repository.save(agg);
  for (var e : agg.pullEvents()) {
    outboxTable.insert(e); // 与 save 同事务
  }
}
// 异步轮询 outbox 表 -> 投递 MQ -> 标记成功
```

------

## 从需求到代码：团队协作流程

### 事件风暴（Event Storming）

- 拉齐业务流程，用橙色卡片标记“发生了什么”（领域事件）
- 辨识命令、参与者、热点不变量与政策

### 战略设计

- 划定限界上下文，绘制 Context Map，定义上下游与 ACL 方案
- 确定统一语言（术语表与样例句）

### 战术设计

- 定义聚合、不变量、状态机
- 规划领域事件（内外之分）、仓储接口

### 一致性策略

- 聚合内事务、跨上下文 Saga/Outbox、幂等键与重试策略

1. API 契约与数据模型

- 用例级别 API/DTO，读模型（查询投影）与资源授权

### 实现与测试

- 先写领域单测（规则/状态机），再写应用服务与集成测试
- 合同测试对齐上下游

### 验收与观测

- 指标、日志、追踪布设；回放链路验证幂等与一致性

------

## 测试策略 🧪

- 领域层单测：不依赖外部系统，覆盖不变量、边界、状态机
- 应用层单测：命令处理、事务、Saga 分支
- 合同测试：上下游 API/事件契约（Consumer-Driven Contract）
- 集成测试：仓储/消息/ACL 最小闭环
- 性能与容量：以聚合为单位的并发冲突与乐观锁回退
- 属性测试（Property-based）：价格分摊、余额不可为负等性质

------

## 可观测性与运维

- 日志：领域事件日志化（eventName, aggregateId, version, payloadSize）
- 指标：下单成功率、延迟、冲突率（乐观锁重试）、幂等命中率、DLQ 数量
- 追踪：traceId 贯穿 API -> 应用 -> 领域事件 -> MQ -> 下游
- 审计：规则变更、风控决策、预算占用与回退留痕

------

## 版本化与演进

- 事件版本：事件载荷加 version 字段，消费者向后兼容
- Schema 演进：向后兼容的字段新增（NULL 安全）、影子写/读双写
- 聚合拆分：先通过应用层编排与事件解耦，再拆库拆服务
- ADR（架构决策记录）：每个关键决策沉淀一条 ADR（见模板）

------

## 常见反模式与检查清单 ✅

反模式：

- 贫血模型：所有规则在应用层，领域层只有 getter/setter
- 巨无霸聚合：跨多个业务能力，导致高冲突与低吞吐
- 直接远程调用替代事件：耦合时序，故障放大
- 值对象可变：引入隐形共享与并发 bug
- 跨聚合强一致：大事务导致锁冲突与可用性下降

检查清单（建模）：

- 聚合是否有清晰的不变量与状态机？
- 跨聚合引用是否仅用 ID？
- 是否区分了领域事件与集成事件？是否版本化？
- 是否使用 Outbox？消费者是否幂等？
- 值对象是否不可变并封装自校验？
- 应用服务是否“只编排，不规则”？

------

## 贡献指南与 ADR 模板

开发约定：

- 新增用例先补齐：用例说明、统一语言更新、聚合不变量清单、测试计划
- 任何跨上下文的集成变更，必须更新 Context Map 与契约文档
- 事件新增需评审：命名、载荷、版本策略、保留周期
- 变更记录：提交信息包含 [context] [use-case] [adr-id]

ADR 模板（adr/adr-YYYYMMDD-标题.md）：

```text
# 背景
问题与约束条件

# 决策
做了什么决策，选型/模式/边界

# 备选方案
优缺点对比

# 后果
对一致性、性能、演进、风险与回滚方案的影响

# 相关
链接到 PR、Issue、文档、契约
```

------

## FAQ

- 什么时候不该用 DDD？
  - 小型纯 CRUD、规则简单、生命周期短的系统，DDD 的成本可能大于收益。
- CQRS 必须引入吗？
  - 非必须。先做清晰的聚合建模，再按查询压力与复杂度引入读模型。
- 一定要微服务吗？
  - 非必须。可先在单体内按上下文模块化，成熟后再拆分服务。

------

如需，我可以基于你的具体业务（如营销、交易、清结算等）补充：

- 上下文地图示例与交互协议
- 聚合类图与不变量表
- Saga 时序图与补偿策略
- Outbox 表结构与投递器实现
- 读模型/索引与缓存策略建议