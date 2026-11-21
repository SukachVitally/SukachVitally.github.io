---
title: "5 Exception Handling Mistakes That Make Java Production Incidents Harder to Debug"
date: 2026-11-21T10:00:00+00:00
url: "/posts/5-exception-handling-mistakes-that-make-java-production-incidents-harder-to-debug/"
draft: false
tags: ["java", "spring-boot", "exceptions", "debugging", "logging", "production"]
images: ["img/image-01.jpg"]
summary: "Log-and-rethrow, lost causes, exceptions on hot paths, catch (Exception) and leaked resources: five habits that turn a 10-minute fix into a 3-hour incident."
---


*The bug took ten minutes to fix. Finding it took three hours.*

![A sleepy cartoon bear in pyjamas and an on-call headset sits at a desk at 3 a.m., shining a torch on a long stack-trace scroll spilling out of the monitor. The scroll's bottom end, labelled "Caused by:", has been torn off and is floating out of the window. Three identical parrots on a shelf squawk "ERROR!" in unison, and a paper aeroplane labelled "email" drifts away into the night unnoticed.](img/image-01.jpg)

It's 3 a.m. and the payments alert has been firing for twenty minutes. The logs show `PaymentException: Charge failed: null` three times, with three different stack traces, none of them pointing below the service layer. The real cause, a TLS handshake failure on the card provider's new endpoint, had been turned into a string along the way, and the string was `null`. You end up reproducing the failure from scratch on a staging box while orders time out. The bug wasn't clever; the exception handling around it just threw away every clue.

This is the fourth article in my "5 Mistakes" series, after [PostgreSQL](/posts/5-postgresql-mistakes-i-see-in-every-spring-boot-app/), [Spring Boot](/posts/5-spring-boot-mistakes-i-see-in-every-codebase/) and [JPA/Hibernate](/posts/5-jpa-hibernate-mistakes-that-quietly-kill-your-spring-boot-performance/). These five come up in almost every code review I do, and each one makes the next incident slower to debug. Examples use Java 21 and Spring Boot 3.x; imports are omitted for brevity.

## Mistake 1: Swallowing exceptions, or logging and rethrowing

![Three bear clerks sit at stacked desks labelled Repository, Service and Controller. Each one stamps "ERROR" on a copy of the same form and passes it up, so the on-call bear at the top is buried under three identical copies. In the corner, a fourth bear quietly feeds a letter marked "timeout" into a shredder labelled "Optional.empty()".](img/image-02.jpg)

**What happened.** An order fails because the inventory reservation hits a lock timeout. The on-call engineer sees three ERROR entries with three full stack traces in the same second, the error-rate alert counts three failures instead of one, and the log search for "Order placement failed" returns the middle one, which is missing the SQL context.

**❌ Bad: every layer logs, then throws**

```java
@Repository
class InventoryRepository {
    private static final Logger log = LoggerFactory.getLogger(InventoryRepository.class);
    private final JdbcClient jdbc;

    InventoryRepository(JdbcClient jdbc) { this.jdbc = jdbc; }

    int reserve(String sku, int qty) {
        try {
            return jdbc.sql("""
                    UPDATE inventory SET reserved = reserved + :qty
                    WHERE sku = :sku AND available - reserved >= :qty""")
                .param("qty", qty).param("sku", sku)
                .update();
        } catch (DataAccessException e) {
            log.error("Failed to reserve stock for {}", sku, e);   // log #1
            throw e;
        }
    }
}

@Service
class OrderService {
    // ...
    Order place(PlaceOrder cmd) {
        try {
            inventory.reserve(cmd.sku(), cmd.quantity());
            return orders.save(Order.from(cmd));
        } catch (Exception e) {
            log.error("Order placement failed", e);                 // log #2
            throw new OrderPlacementException("Order placement failed", e);
        }
    }
}

@RestController
class OrderController {
    // ...
    @PostMapping("/orders")
    ResponseEntity<Order> place(@RequestBody PlaceOrder cmd) {
        try {
            return ResponseEntity.ok(orderService.place(cmd));
        } catch (OrderPlacementException e) {
            log.error("POST /orders failed", e);                    // log #3
            return ResponseEntity.internalServerError().build();
        }
    }
}
```

The opposite mistake is swallowing the exception: catching it and carrying on without logging or rethrowing anything.

```java
Optional<User> findUser(long id) {
    try {
        return Optional.of(userClient.fetch(id));
    } catch (Exception e) {
        return Optional.empty();   // timeout, 500, auth failure: all become "user not found"
    }
}
```

**Why it hurts.** Log-and-rethrow multiplies one failure into N log entries, and every one of them looks like a separate incident. Alerting based on error counts fires three times as often, so people learn to ignore it. Log volume (and the bill for it) grows with the depth of your call stack. The swallow is worse: a timeout from the user service turns into "user not found", the caller returns a 404, and nothing at all appears in the logs. You find out from a customer.

![Diagram. Left, log-and-rethrow: one exception bubbles up from Repository to Service to Controller, and each layer writes its own ERROR entry with a stack trace, so one failure produces 3 log entries and 3 alerts. Right, log once at the boundary: the exception bubbles up through Repository, Service and Controller untouched, and only the @RestControllerAdvice handler logs a single ERROR entry with the full cause chain.](img/diagram-log-and-rethrow.jpg)

**✅ Fixed: let it propagate, log once at the boundary**

```java
@Repository
class InventoryRepository {
    private final JdbcClient jdbc;

    InventoryRepository(JdbcClient jdbc) { this.jdbc = jdbc; }

    boolean reserve(String sku, int qty) {
        return jdbc.sql("""
                UPDATE inventory SET reserved = reserved + :qty
                WHERE sku = :sku AND available - reserved >= :qty""")
            .param("qty", qty).param("sku", sku)
            .update() == 1;
    }
}

@Service
class OrderService {
    // ...
    @Transactional
    Order place(PlaceOrder cmd) {
        if (!inventory.reserve(cmd.sku(), cmd.quantity())) {
            throw new OutOfStockException(cmd.sku(), cmd.quantity());
        }
        return orders.save(Order.from(cmd));
    }
}

// Controller has no try/catch. The global handler (see the bonus section) logs once.

Optional<User> findUser(long id) {
    try {
        return Optional.of(userClient.fetch(id));
    } catch (HttpClientErrorException.NotFound e) {
        return Optional.empty();   // the one case we actually know how to handle
    }
}
```

If a layer has nothing to add, it shouldn't catch. If it does have something to add (an order ID, a SKU), it wraps and rethrows without logging, and the boundary logs the whole chain.

> **Rule:** Handle it or rethrow it, never both. Log once, at the boundary.

## Mistake 2: Losing the root cause

![A bear detective with a magnifying glass opens a large nesting doll labelled "PaymentException" and finds it empty except for a note reading "null". The smaller dolls, labelled "ResourceAccessException" and "SSLHandshakeException", have rolled away under a desk behind the detective.](img/image-03.jpg)

**What happened.** Payments start failing after the provider rotates certificates. The log says `PaymentException: Charge failed: I/O error on POST request for "https://psp.example.com/charges": null`, and the stack trace ends at `PaymentService.java:42`. There is no `Caused by:` section, so nobody can tell whether it was DNS, a timeout, a TLS failure or a refused connection.

**❌ Bad: the cause turned into a string**

```java
PaymentResult charge(ChargeRequest request) {
    try {
        return pspClient.charge(request);
    } catch (RestClientException e) {
        log.warn("Charge failed: " + e.getMessage());               // no stack trace
        throw new PaymentException("Charge failed: " + e.getMessage()); // no cause
    }
}
```

**Why it hurts.** The new exception's stack trace starts at the line where you created it. Everything below that point, including the exception that actually explains the failure, is gone. `getMessage()` is often `null` or a vague summary written by a library author who didn't know your context. The `log.warn` line has the same problem: string concatenation calls `toString()`, which prints one line and no trace.

![Diagram. Top, lost cause: PaymentException "Charge failed: I/O error … null" with cause = null and a stack trace that ends in PaymentService line 42. Bottom, preserved cause: PaymentException for order ord-1042 links via getCause() to ResourceAccessException "I/O error on POST", which links to SSLHandshakeException "Remote host terminated the handshake", the root cause you actually need.](img/diagram-cause-chain.jpg)

**✅ Fixed: add context, keep the cause**

```java
public final class PaymentException extends RuntimeException {
    private final String orderId;

    public PaymentException(String orderId, String message, Throwable cause) {
        super(message, cause);
        this.orderId = orderId;
    }

    public String orderId() { return orderId; }
}

PaymentResult charge(ChargeRequest request) {
    try {
        return pspClient.charge(request);
    } catch (RestClientException e) {
        throw new PaymentException(request.orderId(),
            "Charge failed for order %s, amount %s".formatted(request.orderId(), request.amount()),
            e);
    }
}
```

Now the log shows your message with the order ID, then `Caused by: ResourceAccessException`, then `Caused by: SSLHandshakeException`. When you do log, pass the exception as the last argument: `log.warn("Charge failed for order {}", orderId, e)`. SLF4J treats a trailing `Throwable` as the exception even when there are placeholders, and prints the full chain.

> **Rule:** Every wrap passes the cause. Every log call passes the exception, not its message.

## Mistake 3: Exceptions as control flow on hot paths

![A busy shop till during a sale. Each time a customer asks for an out-of-stock item, the bear cashier pulls a fire alarm, and a team of bears rushes in to unroll an enormous stack-trace scroll before the cashier can say "we don't have it". The queue stretches out of the door under a clock labelled "p99".](img/image-04.jpg)

**What happened.** The basket page checks availability for every item, and around 40% of lookups are for SKUs that have been delisted. During a sale, p99 latency on `/basket` doubles. The profiler shows a large slice of CPU in `Throwable.fillInStackTrace`, called from a `ProductNotFoundException` that nobody ever logs.

**❌ Bad: throwing for an expected outcome, in a loop**

```java
Product getBySku(String sku) {
    return products.findBySku(sku)
        .orElseThrow(() -> new ProductNotFoundException(sku));
}

Map<String, Integer> availability(List<String> skus) {
    var result = new HashMap<String, Integer>();
    for (var sku : skus) {
        try {
            result.put(sku, getBySku(sku).stock());
        } catch (ProductNotFoundException e) {
            result.put(sku, 0);
        }
    }
    return result;
}
```

**Why it hurts.** Creating a `Throwable` calls `fillInStackTrace()`, which walks the entire current stack. In a Spring Boot request that's easily 100 to 200 frames of filters, proxies and interceptors, so each "not found" costs far more than the lookup that produced it. The JIT can sometimes optimise a throw and catch in the same compiled method, but not across a repository call and a proxy. On top of that, this code does N queries instead of one.

There's a debugging trap nearby, too. For implicit exceptions thrown very often from the same spot (`NullPointerException`, `ArithmeticException`, `ArrayIndexOutOfBoundsException`), HotSpot eventually switches to a preallocated instance with no stack trace at all. Your logs then show a bare `java.lang.NullPointerException` with nothing under it. `-XX:-OmitStackTraceInFastThrow` turns that optimisation off, and it's worth setting in production.

**✅ Fixed: model the expected case as data**

```java
Map<String, Integer> availability(List<String> skus) {
    var stockBySku = products.findAllBySkuIn(skus).stream()
        .collect(Collectors.toMap(Product::sku, Product::stock));
    return skus.stream()
        .distinct()
        .collect(Collectors.toMap(sku -> sku, sku -> stockBySku.getOrDefault(sku, 0)));
}
```

If an exception really must signal a frequent, expected outcome, for example inside a parser or rule engine where unwinding many frames at once is the point, switch the stack trace off:

```java
public final class StockUnavailableException extends RuntimeException {
    public StockUnavailableException(String sku) {
        super("No stock for " + sku, null, false, false); // no suppression, no stack trace
    }
}
```

That makes it cheap, but if one escapes into your logs you'll have no idea where it came from. Use it only for exceptions that are always caught close to where they're thrown.

> **Rule:** If it happens on every tenth request, it isn't exceptional. Return it, don't throw it.

## Mistake 4: Catching Exception or Throwable too broadly

![A bear worker keeps running on a hamster wheel labelled "while (true)" while wearing earmuffs labelled "catch (Exception e)". A supervisor bear shouts "Shutdown! Time to go home!" through a megaphone, a wall clock counts down "30 s", and a giant hand marked "SIGKILL" reaches in through the doorway.](img/image-05.jpg)

**What happened.** A deployment rolls out a new version of the notification service. Kubernetes sends SIGTERM, Spring starts shutting down, and then nothing happens for 30 seconds until the pod is killed. In-flight emails are sent twice after restart. A thread dump from the next rollout shows the dispatcher thread still looping.

**❌ Bad: catch everything, keep going**

```java
class NotificationDispatcher implements Runnable {
    // ...
    @Override
    public void run() {
        while (true) {
            try {
                var batch = outbox.fetchPending(100);
                batch.forEach(sender::send);
                if (batch.isEmpty()) {
                    Thread.sleep(1_000);
                }
            } catch (Exception e) {
                log.error("Dispatch failed, retrying", e);
            }
        }
    }
}
```

**Why it hurts.** Shutting down an executor (`shutdownNow()`, `close()`, or Spring destroying a `ThreadPoolTaskExecutor`) works by interrupting its threads. When `Thread.sleep` throws `InterruptedException`, the interrupt flag is already cleared. `catch (Exception e)` catches it, logs it as an error, and the loop carries on as if nothing happened, so the thread can only stop by being killed. The same catch also turns a programming bug, such as an NPE on a malformed notification, into an endless retry loop that writes a stack trace per iteration. `catch (Throwable t)` is worse: it also catches `OutOfMemoryError` and `StackOverflowError`, after which the JVM's state can't be trusted. One more in Spring: catching inside a `@Transactional` method and not rethrowing means the transaction commits whatever was half-written.

**✅ Fixed: catch what you can handle, respect interrupts**

```java
class NotificationDispatcher implements Runnable {
    private static final Logger log = LoggerFactory.getLogger(NotificationDispatcher.class);
    private static final Duration IDLE = Duration.ofSeconds(1);
    private static final Duration BACK_OFF = Duration.ofSeconds(5);
    // ...

    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                var batch = outbox.fetchPending(100);
                batch.forEach(this::dispatch);
                if (batch.isEmpty()) {
                    pause(IDLE);
                }
            } catch (TransientDataAccessException e) {
                log.warn("Outbox temporarily unavailable, backing off", e);
                pause(BACK_OFF);
            }
        }
        log.info("Notification dispatcher stopped");
    }

    private void dispatch(Notification n) {
        try {
            sender.send(n);
            outbox.markSent(n.id());
        } catch (DeliveryException e) {
            log.warn("Delivery failed for notification {}", n.id(), e);
            outbox.markFailed(n.id());
        }
    }

    private static void pause(Duration duration) {
        try {
            Thread.sleep(duration);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore the flag, the loop checks it
        }
    }
}
```

Known, recoverable failures are handled where they make sense. Anything else (a bug, an `Error`) ends the loop and propagates, instead of being retried forever. Mistake 5 covers making sure someone actually sees it. `catch (Exception e)` does have a place: the very top of a thread, a scheduled job or a message listener, where you log once and stop or move on. Even there, never catch `Throwable` except to log it and rethrow.

> **Rule:** Catch only what you can handle. If you catch `InterruptedException`, restore the flag or rethrow.

## Mistake 5: Letting exceptions disappear in async code

![A bear at an open window cheerfully launches paper aeroplanes labelled "email" and "loyalty points" into the fog. Several of them crash into a growing pile below labelled "Future" that nobody is watching. On the desk behind the bear, the open log book has only blank pages, and outside, customers stand next to empty mailboxes.](img/image-06.jpg)

**What happened.** After an order is placed, the confirmation email and the loyalty-points update run in the background. For two days a growing share of customers get no email, and support finds out before monitoring does. The logs have nothing: no ERROR, no stack trace, not even a warning. The SMTP credentials had been rotated, and every failure was sitting inside a `Future` that nobody ever looked at.

**❌ Bad: fire and forget**

```java
@Service
class OrderConfirmationService {
    private final ExecutorService executor;
    private final EmailSender emailSender;
    private final LoyaltyClient loyalty;
    // ...

    void onOrderPlaced(Order order) {
        executor.submit(() -> emailSender.sendConfirmation(order));          // Future ignored
        CompletableFuture.runAsync(
            () -> loyalty.addPoints(order.userId(), order.total()), executor); // result ignored
    }
}
```

**Why it hurts.** `submit()` wraps the task in a `FutureTask`, which catches whatever the task throws and stores it in the `Future`. It is reported only when someone calls `get()`. The thread's uncaught-exception handler never sees it, so nothing is logged anywhere. `CompletableFuture` works the same way: the exception completes the future and waits there to be joined or handled. Spring's `@Async` methods that return a `CompletableFuture` behave identically, and only `void` `@Async` methods reach the `AsyncUncaughtExceptionHandler`. Unlike every other mistake in this article, this one doesn't give you a bad stack trace. It gives you no stack trace at all.

Even when an async failure does get logged, it often arrives without context. MDC values and the trace ID belong to the request thread, so the ERROR line from the pool thread can't be linked to the request that caused it.

**✅ Fixed: every async task has someone who handles its failure**

```java
@Service
class OrderConfirmationService {
    private static final Logger log = LoggerFactory.getLogger(OrderConfirmationService.class);
    private final Executor notificationExecutor;
    private final EmailSender emailSender;
    private final LoyaltyClient loyalty;
    // ...

    void onOrderPlaced(Order order) {
        runAsync("confirmation email", order, () -> emailSender.sendConfirmation(order));
        runAsync("loyalty points", order, () -> loyalty.addPoints(order.userId(), order.total()));
    }

    private void runAsync(String task, Order order, Runnable action) {
        CompletableFuture.runAsync(action, notificationExecutor)
            .whenComplete((ignored, e) -> {
                if (e != null) {
                    log.error("Async {} failed for order {}", task, order.id(), e);
                }
            });
    }
}

@Configuration
class AsyncConfig {
    @Bean
    ThreadPoolTaskExecutor notificationExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setThreadNamePrefix("notify-");
        // copies MDC / trace context to the worker thread (Spring 6.1+, needs io.micrometer:context-propagation)
        executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
        return executor;
    }
}
```

If the caller needs the result, return the `CompletableFuture` and let the caller join it, so failures flow back to the request and the global handler. If it's genuinely fire-and-forget, the task logs its own failure, because nobody else ever will. For anything that must not be lost, such as a confirmation email, an outbox table with retries is better than a thread pool, but that's a topic for another article.

> **Rule:** Every async task needs an owner for its failure: join it, handle it, or log it inside the task.

## Bonus: what a good exception strategy looks like

Most of the fixes above point to the same small set of decisions.

**Two kinds of exceptions.** *Domain exceptions* are business outcomes: order not found, out of stock, payment declined. They're expected, they carry data (an order ID, a SKU), they map to 4xx, and they don't need a stack trace in the logs. *Technical exceptions* are infrastructure failures: timeouts, lock waits, TLS errors. They map to 5xx, and you want the full cause chain, logged once.

A sealed hierarchy makes the domain side explicit, and the compiler checks your mapping:

```java
public abstract sealed class DomainException extends RuntimeException
        permits OrderNotFoundException, OutOfStockException, PaymentDeclinedException {
    protected DomainException(String message) { super(message); }
}

public final class OrderNotFoundException extends DomainException {
    public OrderNotFoundException(long orderId) { super("Order %d not found".formatted(orderId)); }
}

public final class OutOfStockException extends DomainException {
    public OutOfStockException(String sku, int requested) {
        super("Not enough stock for %s (requested %d)".formatted(sku, requested));
    }
}

public final class PaymentDeclinedException extends DomainException {
    public PaymentDeclinedException(String orderId, String reason) {
        super("Payment for order %s declined: %s".formatted(orderId, reason));
    }
}
```

**Log once, at the boundary.** For HTTP that's a `@RestControllerAdvice`. Extending `ResponseEntityExceptionHandler` keeps Spring MVC's own exceptions (malformed JSON, missing parameters) as proper 400s with `ProblemDetail`, instead of letting a catch-all turn them into 500s.

```java
@RestControllerAdvice
class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(DomainException.class)
    ProblemDetail handleDomain(DomainException e) {
        var status = switch (e) {
            case OrderNotFoundException ignored -> HttpStatus.NOT_FOUND;
            case OutOfStockException ignored -> HttpStatus.CONFLICT;
            case PaymentDeclinedException ignored -> HttpStatus.UNPROCESSABLE_ENTITY;
        };
        log.info("Request rejected: {}", e.getMessage()); // expected outcome, no stack trace
        var problem = ProblemDetail.forStatusAndDetail(status, e.getMessage());
        problem.setTitle(status.getReasonPhrase());
        return problem;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception e, HttpServletRequest request) {
        var errorId = UUID.randomUUID().toString();
        log.error("Unhandled error {} on {} {}",
            errorId, request.getMethod(), request.getRequestURI(), e); // the one ERROR, full chain
        var problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "Unexpected error. Reference: " + errorId);
        problem.setProperty("errorId", errorId);
        return problem;
    }
}
```

The switch is exhaustive because `DomainException` is sealed, so adding a fourth domain exception without a status mapping is a compile error. The client gets a stable RFC 9457 body, and the `errorId` it reports to support takes you straight to the one log line that matters. If you run Micrometer Tracing, put the trace ID there instead.

HTTP isn't the only boundary. `@Scheduled` jobs, Kafka listeners, `@Async` methods and background `CompletableFuture`s each need their own single place that logs: an error handler, an `AsyncUncaughtExceptionHandler`, a `whenComplete` callback, or a try/catch at the very top of the job.

## Code review checklist

- Does any `catch` block both log and rethrow? Pick one.
- Does every exception created inside a `catch` pass the caught exception as its cause?
- Do log calls in `catch` blocks pass `e` as the last argument, rather than `e.getMessage()` or string concatenation?
- Is an exception thrown and caught for an expected outcome inside a loop or on a hot path?
- Is there a `catch (Exception | Throwable)` anywhere other than a top-level boundary? Is any `InterruptedException` caught without `interrupt()` or a rethrow?
- Does every `submit()`, `CompletableFuture` and non-`void` `@Async` call have someone who joins it or handles its failure?
- Is there exactly one global handler that returns `ProblemDetail` and logs unexpected errors once, with the full cause chain?

## Conclusion

None of these mistakes cause the incident. They decide how long it lasts. A preserved cause chain, one log entry per failure and threads that stop when asked are what let the on-call engineer go from alert to root cause in minutes, at 3 a.m., without reproducing anything.

Pick one item from the checklist and grep your codebase for it this week. `catch (Exception` and `e.getMessage())` are good places to start.

If this was useful, follow me for the next article in the "5 Mistakes" series.
