---
title: "5 REST API Design Mistakes I Still See in Spring Boot Code Reviews"
date: 2026-11-26T10:00:00+00:00
url: "/posts/5-rest-api-design-mistakes-i-still-see-in-spring-boot-code-reviews/"
draft: false
tags: ["rest", "api-design", "spring-boot", "java", "http", "idempotency"]
images: ["img/image-01.jpg"]
summary: "RPC-style URLs, lying status codes, entities as responses, unbounded lists and retry-unsafe POSTs: 5 API mistakes that pass every test and fail in production."
---

*Your API works today. These five mistakes are why it'll hurt in a year.*

![A cartoon bear tech lead with a clipboard labelled "Code review" inspects a vending machine labelled "API" that is misbehaving in five ways at once: a GET button with a note reading "cancels order", a display showing a cheerful "200 OK" while smoke pours out of the back, a whole filing cabinet squeezing out of the delivery slot, an endless ribbon of receipts unrolling across the floor, and two identical parcels dropping out where one was ordered.](img/image-01.jpg)

API design mistakes don't fail on day one. They fail when something else starts depending on your API: a proxy that caches GETs, a client that retries, a partner that pages through 200,000 orders. By then the contract is public, and every fix is a breaking change.

This is the next article in my "5 Mistakes" series, after [PostgreSQL](/posts/5-postgresql-mistakes-i-see-in-every-spring-boot-app/), [Spring Boot](/posts/5-spring-boot-mistakes-i-see-in-every-codebase/), [JPA/Hibernate](/posts/5-jpa-hibernate-mistakes-that-quietly-kill-your-spring-boot-performance/) and [exception handling](/posts/5-exception-handling-mistakes-that-make-java-production-incidents-harder-to-debug/). Examples use an online shop, Java 21 and Spring Boot 3.x; imports are omitted.

## Mistake 1: RPC-style URLs and ignoring HTTP semantics

![In a mail room, a small robot labelled "Link scanner" holds a magnifying glass and points at a red button reading "GET /orders/42/cancel" in an order email on a monitor, next to buttons for viewing and tracking the order. An order box tumbles into a bin marked "Cancelled", while a bear holding a coffee mug clutches its head in shock.](img/image-02.jpg)

**What happened.** The order confirmation email linked to `GET /api/orders/42/cancel`. The customer's corporate email scanner opened every link to check it for malware, and the order was cancelled before the customer had read the email.

**❌ Bad: verbs in URLs, GET with side effects**

```java
@GetMapping("/api/getOrder")
OrderResponse getOrder(@RequestParam long id) { return orderService.find(id); }

@PostMapping("/api/orders/delete/{id}")
void deleteOrder(@PathVariable long id) { orderService.delete(id); }

@GetMapping("/api/orders/{id}/cancel")   // linked from the confirmation email
String cancelOrder(@PathVariable long id) {
    orderService.cancel(id);
    return "Order cancelled";
}
```

**Why it hurts.** The HTTP method is a contract that infrastructure you don't control relies on. [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-method-properties) defines GET, HEAD and OPTIONS as *safe*: they don't ask for any state change. So browsers prefetch them, crawlers and link scanners follow them, and caches store them. A GET that cancels an order will be called by something other than your customer.

PUT and DELETE are *idempotent*: sending one twice has the same effect as sending it once, so clients and proxies may retry them automatically after a dropped connection. They won't retry a POST. `POST /orders/delete/42` takes an operation that's naturally safe to retry and makes it look unsafe.

**✅ Better: resources as nouns, the method as the verb**

```java
@GetMapping("/api/orders/{id}")
OrderResponse get(@PathVariable long id) { return orderService.find(id); }

@DeleteMapping("/api/orders/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
void delete(@PathVariable long id) { orderService.delete(id); }

@PostMapping("/api/orders/{id}/cancellation")
OrderResponse cancel(@PathVariable long id) { return orderService.cancel(id); }
```

Don't force every business action into CRUD. Cancelling has rules (you can't cancel a shipped order), so a POST that creates a `cancellation` sub-resource is honest. The email link should open a page with a button that sends that POST.

> **Takeaway:** GET must never change state. Caches, scanners and retrying clients take the method at its word.

## Mistake 2: Lying status codes

![A relaxed bear in a headset leans back in front of a big dashboard showing "System Health 99.98% OK" in green. Through the window behind it, a pile of boxes marked "checkout failed" is on fire, each with a sticky note reading "200 OK". In the corner, an on-call bear in pyjamas is jolted awake by a pager showing "500: postcode typo".](img/image-03.jpg)

**What happened.** The dashboard showed 99.98% success while a third of checkouts failed, because errors came back as `200 OK` with `{"success": false}`. The same week, a typo in a postcode returned `500` and paged the on-call engineer at 2 a.m.

**❌ Bad: errors disguised as success, client mistakes reported as server failures**

```java
@PostMapping("/api/orders")
ResponseEntity<Map<String, Object>> create(@RequestBody CreateOrderRequest request) {
    try {
        var order = orderService.create(request);
        return ResponseEntity.ok(Map.of("success", true, "orderId", order.id()));
    } catch (OutOfStockException e) {
        return ResponseEntity.ok(Map.of("success", false, "error", e.getMessage()));
    } catch (IllegalArgumentException e) {   // "quantity must be positive"
        return ResponseEntity.internalServerError().body(Map.of("error", e.getMessage()));
    }
}
```

**Why it hurts.** Error-rate dashboards, SLOs, gateways and client retry policies read the status line, not your body. A `200` with an error inside is invisible to all of them. A `500` for bad input is the opposite problem: clients and gateways retry it, your error budget burns, and someone gets paged for a user's typo.

The codes you need most of the time:

- **400 Bad Request**: malformed JSON or failed bean validation.
- **404 Not Found**: the order doesn't exist, or the caller mustn't know it does.
- **409 Conflict**: clashes with the current state, such as cancelling a shipped order.
- **422 Unprocessable Content**: well formed but breaks a business rule, such as ordering more than is in stock. Spring 6 still calls it `UNPROCESSABLE_ENTITY`, its pre-RFC 9110 name.

**✅ Better: honest status codes with an RFC 9457 body**

Spring 6 supports [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) through `ProblemDetail`. In Spring Boot 3, `spring.mvc.problemdetails.enabled=true` makes Spring MVC's own errors (validation, unreadable JSON, missing parameters) use it. For domain exceptions, add an advice. If it extends `ResponseEntityExceptionHandler`, it covers the built-in cases too, and Boot's default handler backs off.

```java
@RestControllerAdvice
class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(OrderStateException.class)
    ProblemDetail handleState(OrderStateException e) {
        return problem(HttpStatus.CONFLICT, "invalid-order-state", "Order cannot be changed", e.getMessage());
    }

    @ExceptionHandler(OutOfStockException.class)
    ProblemDetail handleOutOfStock(OutOfStockException e) {
        var problem = problem(HttpStatus.UNPROCESSABLE_ENTITY, "out-of-stock", "Product out of stock", e.getMessage());
        problem.setProperty("productId", e.productId());
        problem.setProperty("available", e.available());
        return problem;
    }

    private static ProblemDetail problem(HttpStatus status, String type, String title, String detail) {
        var problem = ProblemDetail.forStatusAndDetail(status, detail);
        problem.setType(URI.create("https://api.example-shop.com/problems/" + type));
        problem.setTitle(title);
        return problem;
    }
}
```

The client gets a `422` with `Content-Type: application/problem+json`:

```json
{
  "type": "https://api.example-shop.com/problems/out-of-stock",
  "title": "Product out of stock",
  "status": 422,
  "detail": "Only 2 units of product 1042 left, 5 requested",
  "instance": "/api/orders",
  "productId": 1042,
  "available": 2
}
```

Clients switch on `type`, humans read `detail`, and the extra properties carry what a client needs to recover. Spring fills in `instance`. The [exception handling article](/posts/5-exception-handling-mistakes-that-make-java-production-incidents-harder-to-debug/) covers the logging side of this handler.

> **Takeaway:** 2xx means it worked, 4xx means the client must change something, 5xx means you broke something.

## Mistake 3: Returning JPA entities from controllers

![At a shop service window, a customer bear has asked for a small card showing just a name and an email. The clerk bear is pushing an entire filing cabinet labelled "Customer entity" through the window, with drawers spilling papers marked "passwordHash", "vip" and a thick bundle labelled "orders × 10,000". The customer looks horrified.](img/image-04.jpg)

**What happened.** Someone added a list of orders to the `Customer` entity for an unrelated report. `GET /api/customers/{id}` returned the entity, so its response grew from 1 KB to 4 MB overnight. When Open Session in View was later switched off, the endpoint started throwing `LazyInitializationException`.

**❌ Bad: the entity is the API**

```java
@Entity
class Customer {
    @Id @GeneratedValue Long id;
    String email;
    String passwordHash;
    boolean vip;
    @OneToMany(mappedBy = "customer") List<Order> orders = new ArrayList<>();
    // getters and setters
}

@GetMapping("/api/customers/{id}")
Customer get(@PathVariable long id) { return customers.findById(id).orElseThrow(); }

@PutMapping("/api/customers/{id}")
Customer update(@PathVariable long id, @RequestBody Customer customer) {
    customer.setId(id);
    return customers.save(customer);
}
```

**Why it hurts.** One shortcut, five problems:

- **Lazy loading in Jackson.** With Open Session in View on (Spring Boot's default), serialisation fires a query per lazy association. With it off, you get `LazyInitializationException`. My [JPA/Hibernate article](/posts/5-jpa-hibernate-mistakes-that-quietly-kill-your-spring-boot-performance/) explains why OSIV should be off.
- **Infinite recursion.** `Order.customer` points back to `Customer.orders`, and Jackson follows the loop until it fails.
- **Leaked internals.** `passwordHash` and the `vip` flag are now public API.
- **Mass assignment.** A client sends `"vip": true` and it's saved. One that omits `email` sets it to `null`.
- **Schema coupling.** Renaming a column is now a breaking API change.

**✅ Better: API records at the edge, plain DTOs in the service, mappers in between**

```java
// API layer: the public contract
public record UpdateCustomerRequest(
        @NotBlank @Email String email,
        @NotBlank @Size(max = 100) String displayName) {}

public record CustomerResponse(long id, String email, String displayName, OffsetDateTime createdAt) {}

// service layer: knows nothing about HTTP
public record UpdateCustomerDto(String email, String displayName) {}

public record CustomerDto(long id, String email, String displayName, OffsetDateTime createdAt) {}

@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface CustomerApiMapper {
    UpdateCustomerDto toDto(UpdateCustomerRequest request);
    CustomerResponse toResponse(CustomerDto customer);
}

@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface CustomerMapper {
    CustomerDto toDto(Customer customer);
}

@PutMapping("/api/customers/{id}")   // in CustomerController
CustomerResponse update(@PathVariable long id, @Valid @RequestBody UpdateCustomerRequest request) {
    var updated = customerService.update(id, apiMapper.toDto(request));
    return apiMapper.toResponse(updated);
}

@Transactional
public CustomerDto update(long id, UpdateCustomerDto changes) {   // in CustomerService
    var customer = customers.findById(id).orElseThrow(() -> new CustomerNotFoundException(id));
    customer.changeEmail(changes.email());
    customer.rename(changes.displayName());
    return customerMapper.toDto(customer);
}
```

Each layer has its own types. The request record lists exactly what a client may change, the response record exactly what it may see, and the service works only with its own DTOs. A new API version or a renamed JSON field changes the controller and its mapper, not the service. All the records are plain data. [MapStruct](https://mapstruct.org/) generates the mapping code at compile time, so there's no reflection at runtime, and a field it can't map shows up as a build warning, not a silent `null`. The entity is mapped inside the transaction, so lazy data is loaded on purpose, not by Jackson.

> **Takeaway:** Entities model your database, DTOs model your contract. Don't let one stand in for the other.

## Mistake 4: Unbounded collections and naive pagination

![Split scene in a warehouse library. On the left, a forklift bear dumps a mountain of 200,000 order boxes onto a tiny delivery van labelled "GET /orders", whose tyres are bursting. On the right, two librarian bears look for page 5,001: one tiredly counts every book from shelf 1 under a sign reading "OFFSET", while the other opens straight to a bookmark labelled "cursor" and smiles.](img/image-05.jpg)

**What happened.** `GET /api/orders?customerId=…` returned every order for the account: 300 rows in staging, 200,000 for one business customer in production. The response took 40 seconds and the heap spike took the pod down. The client timed out and retried, twice.

**❌ Bad: a Page of entities, serialised as-is**

```java
@GetMapping("/api/orders")
Page<Order> list(@RequestParam long customerId, Pageable pageable) {
    return orders.findByCustomerId(customerId, pageable);
}
```

**Why it hurts.** Adding `Pageable` is the usual first fix, and it still has problems:

- It returns entities (see Mistake 3).
- Since Spring Data 3.3, serialising `PageImpl` directly logs a warning that its JSON structure isn't guaranteed to stay stable.
- `size` comes from the client, and Spring Data's default cap is 2,000.
- `OFFSET` gets slower the deeper you go. `OFFSET 100000 LIMIT 20` makes the database read and throw away 100,000 rows to return 20, and new orders shift rows between pages, so clients see duplicates or gaps.

![Diagram. OFFSET pagination: for page 5,001 the database walks the index from the start, reads 100,000 rows, discards them and returns 20, so cost grows with page depth. Keyset pagination: the database seeks directly to the last seen (created_at, id) position in the index and reads the next 20 rows, so cost stays constant at any depth.](img/diagram-pagination.jpg)

**✅ Better: a hard limit and a stable format, plus a cursor for big tables**

For admin screens, bounded offset pagination is fine:

```properties
spring.data.web.pageable.max-page-size=100
```

```java
@Configuration
@EnableSpringDataWebSupport(pageSerializationMode = EnableSpringDataWebSupport.PageSerializationMode.VIA_DTO)
class WebConfig {}

@GetMapping("/api/orders")
PagedModel<OrderSummary> list(@RequestParam long customerId, Pageable pageable) {
    return new PagedModel<>(orders.findByCustomerId(customerId, pageable).map(orderMapper::toSummary));
}
```

`PagedModel` (Spring Data 3.3+) gives a stable shape: `content` plus a `page` object with `size`, `number`, `totalElements` and `totalPages`. With `VIA_DTO` on, a plain `Page` return value is wrapped the same way.

For large or growing tables (order history, feeds, anything a client syncs), use keyset pagination. The client sends back an opaque cursor encoding the last row it saw:

```java
interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("""
        select o from Order o
        where o.customer.id = :customerId
          and (o.createdAt < :createdAt or (o.createdAt = :createdAt and o.id < :id))
        order by o.createdAt desc, o.id desc""")
    List<Order> findPageAfter(long customerId, OffsetDateTime createdAt, long id, Limit limit);
}

public record OrderCursor(OffsetDateTime createdAt, long id) {
    static OrderCursor of(Order order) { return new OrderCursor(order.getCreatedAt(), order.getId()); }

    String encode() {
        var raw = createdAt + "|" + id;
        return Base64.getUrlEncoder().withoutPadding().encodeToString(raw.getBytes(StandardCharsets.UTF_8));
    }

    static OrderCursor decode(String cursor) {
        var parts = new String(Base64.getUrlDecoder().decode(cursor), StandardCharsets.UTF_8).split("\\|");
        return new OrderCursor(OffsetDateTime.parse(parts[0]), Long.parseLong(parts[1]));
    }
}

@Transactional(readOnly = true)
public OrderSlice nextPage(long customerId, String cursor, int requestedLimit) {
    var limit = Math.clamp(requestedLimit, 1, 100);
    var after = OrderCursor.decode(cursor);
    var rows = orders.findPageAfter(customerId, after.createdAt(), after.id(), Limit.of(limit + 1));
    var hasMore = rows.size() > limit;
    var page = hasMore ? rows.subList(0, limit) : rows;
    var next = hasMore ? OrderCursor.of(page.getLast()).encode() : null;
    return new OrderSlice(page.stream().map(orderMapper::toSummary).toList(), next);
}
```

The first page is the same query without the cursor condition. Fetching `limit + 1` rows tells you whether there's a next page without a `COUNT(*)`, and the `id` tiebreaker stops rows with equal timestamps being skipped or repeated. With an index on `(customer_id, created_at DESC, id DESC)`, page 5,000 costs the same as page 1. The trade-off is no jumping to page 37, which large lists rarely need. The [PostgreSQL article](/posts/5-postgresql-mistakes-i-see-in-every-spring-boot-app/) covers the query-plan side.

> **Takeaway:** Every collection endpoint needs a hard limit. Large ones need a cursor, not a page number.

## Mistake 5: Non-idempotent POSTs with no retry safety

![A bear on a train taps "Place order" on a phone just as the train enters a tunnel, and the reply envelope labelled "201 Created" flutters away into the dark. At the warehouse, a puzzled bear packs two identical parcels. In the corner, the same parcel, now with a ticket stub reading "Idempotency-Key: 7c1e", is packed only once and gets a thumbs-up.](img/image-06.jpg)

**What happened.** A customer placed an order on a train. The server created it in 300 ms, but the response was lost in a tunnel. The app timed out, retried, and the server created a second order. The warehouse packed two identical parcels.

**❌ Bad: every POST creates something new**

```java
@PostMapping("/api/orders")
@ResponseStatus(HttpStatus.CREATED)
OrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
    return orderService.create(request);
}
```

**Why it hurts.** To the client, a timeout is ambiguous: the request may have failed before it reached you, succeeded with the response lost, or still be running. It can't tell which, so it retries. Retries come from everywhere: double taps, mobile SDKs, gateway and service-mesh retry policies, message replays. Disabling the button covers one of them.

You can't deduplicate on the body either, because a customer can legitimately order the same thing twice. The client has to tell you which requests are the *same attempt*.

![Diagram. Without an idempotency key: the client sends POST /orders, the server creates order 1001 but the response is lost, the client times out and retries, and the server creates order 1002, so one purchase becomes two orders. With an idempotency key: the client sends POST /orders with Idempotency-Key 7c1e, the server stores the key with order 1001 in one transaction, the response is lost, the client retries with the same key, the server finds the key and returns the stored 201 response, so there is exactly one order.](img/diagram-idempotency.jpg)

**✅ Better: an Idempotency-Key header with a stored response**

The pattern is specified in the IETF draft [The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/). It isn't an RFC yet, but many public APIs already use it. The client generates a UUID per *intent* ("place this basket as an order") and sends it with every retry of that intent.

Keys live in a table whose primary key does the real work. Scoping it by customer stops one client replaying another's response:

```sql
CREATE TABLE idempotency_keys (
    customer_id     BIGINT      NOT NULL,
    idempotency_key VARCHAR(64) NOT NULL,
    request_hash    CHAR(64)    NOT NULL,
    response_body   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (customer_id, idempotency_key)
);
```

The controller requires `@RequestHeader("Idempotency-Key")`; a missing header becomes a `400` through the handler from Mistake 2. The service then handles three cases:

1. **Known key:** it's a retry, so nothing is created. If the request hash matches, return the stored response with an `Idempotent-Replayed: true` header. If it doesn't, the client reused a key for a different request: return `422`.
2. **New key:** in one transaction, insert the key row *first*, then create the order and store its response on the key row. A crash halfway rolls back both, so you never get an order without a key, or the other way round.
3. **Unique violation on that insert:** a parallel duplicate got there first. PostgreSQL made this `INSERT` wait on the other transaction's uncommitted row, so the duplicate did no work. Look the key up again in a fresh transaction (PostgreSQL won't run anything else in the failed one) and replay the winner's response. No explicit locks needed.

Only successes are stored: a failed request rolls back, key included, so a retry runs again, and an out-of-stock order may succeed after a restock. Delete keys after, say, 24 hours, and document that window so clients know how long a retry is protected.

This works while everything lives in your database. Once the request calls something external, such as a warehouse API, claim the key in its own committed transaction as `IN_PROGRESS` with a `locked_until` time, do the work, then store the response as `COMPLETED`. A concurrent duplicate that finds `IN_PROGRESS` gets `409` with `Retry-After`. If the server died mid-flight, the next retry takes over once the lock expires. Pass the same key downstream, so the external call is idempotent too.

One rule on the client side: generate the key once per intent and reuse it on every retry. A new UUID per attempt gives you all of this machinery and none of the protection.

> **Takeaway:** Any POST that creates something needs an idempotency key, because the client can't tell a lost response from a failed request.

## Bonus: three smaller things to fix before v1

- **No versioning strategy.** Decide how you'll ship a breaking change before you need to. A `/api/v1` prefix is the most practical choice; additive changes don't need a new version, renames and removals do. Spring Framework 7 (Spring Boot 4) adds built-in API versioning.
- **Inconsistent field naming.** `customerId` here, `customer_id` there, `clientId` somewhere else. Pick one convention, configure Jackson once and lint your OpenAPI spec in CI.
- **Dates without time zones.** `LocalDateTime` serialises as `2026-11-26T10:00:00`, a different instant for every reader. Use `OffsetDateTime` or `Instant` so the JSON carries the offset: `2026-11-26T10:00:00Z`. More in [Datetime Handling in Java and PostgreSQL](/posts/datetime-handling-in-java-and-postgresql-a-practical-guide/).

## Conclusion

None of these mistakes break your tests. They break when the API meets the real world: a link scanner, a dashboard, a big customer, a train tunnel. They're cheap to fix now and expensive once clients depend on them.

Your checklist for the next API review:

- **Methods:** URLs are nouns, the method is the verb, and no GET changes state.
- **Status codes:** errors are 4xx or 5xx, never `200`, with a `ProblemDetail` body.
- **DTOs:** controllers take and return records, never entities.
- **Collections:** every list has a hard limit; big ones use a keyset cursor.
- **Idempotency:** every creating POST takes an `Idempotency-Key` and replays the stored response.

If this was useful, follow me for the next article in the "5 Mistakes" series.
