---
title: "Your AI Assistant Is Just Another Team That Ignores Your Bounded Context"
date: 2026-11-15T10:00:00+00:00
url: "/posts/your-ai-assistant-is-just-another-team-that-ignores-your-bounded-context/"
draft: false
tags: ["ddd", "ai", "spring-boot", "java", "archunit", "architecture"]
summary: "AI assistants cross DDD boundaries like a new team that never read the design docs. Make the domain explicit for the AI, then enforce it with tests."
images: ["img/image-01.jpg"]
---

![A cheerful robot with a "New team – Day 1" badge breaks through the wall of a garden labelled "Loyalty", carrying a glowing jar of points and dropping coins behind it. It ignores the open gate nearby with the sign "LoyaltyService – please use this entrance", and an unread AGENTS.md scroll lies on the ground. Two surprised developers with coffee mugs watch.](img/image-01.jpg)

> **TL;DR**
>
> - An AI assistant takes the shortest path through your code, and that path usually crosses a boundary.
> - It sees code, not intent. If the rules live in people's heads, the AI can't follow them.
> - Write the bounded context down in an `AGENTS.md` next to the code: language, invariants, public API, forbidden dependencies.
> - Prompts are suggestions, tests are guarantees. Enforce the boundary with ArchUnit.

## The AI wrote a feature that works and breaks four rules

The ticket was simple: "Customers get 2x loyalty points on their birthday." An AI assistant opened the checkout module, found the loyalty repository, and produced this:

```java
@Service
public class CheckoutService {

    private final OrderRepository orderRepository;
    private final LoyaltyAccountRepository loyaltyAccountRepository;

    // constructor omitted

    @Transactional
    public void completeCheckout(Order order, Customer customer) {
        orderRepository.save(order);
        LoyaltyAccount account = loyaltyAccountRepository
                .findByCustomerId(customer.getId()).orElseThrow();
        long points = order.getTotal().longValue();
        if (MonthDay.from(customer.getBirthday()).equals(MonthDay.now())) {
            points *= 2;
        }
        account.setBalance(account.getBalance() + points);
        loyaltyAccountRepository.save(account);
    }
}
```

It compiles. The tests pass. The feature works in the demo. Here is what it breaks:

- **Checkout reaches into Loyalty's persistence.** `LoyaltyAccountRepository` is an internal detail of another context.
- **Points before payment.** The order is saved, not paid. A failed card still earns points.
- **Checkout changes the balance itself.** Calling `setBalance()` from checkout skips every rule `LoyaltyService` is supposed to enforce.
- **No event.** `PointsEarned` is never published, so the email service and analytics never hear about it.

![Diagram: CheckoutService in the Checkout context has two paths into the Loyalty context. The intended path goes to LoyaltyService, the public API that returns DTOs. The AI's shortcut, marked with a red cross, goes straight to the hidden LoyaltyAccountRepository and LoyaltyAccount. The rules it breaks: uses another context's repository, points before payment, sets the balance directly, publishes no event.](img/diagram-shortcut.jpg)

None of this is new. A team that joined last month and never read the design docs would write the same code, for the same reason: nobody told them where the boundary was.

## Why the AI crosses the line

**The shortest path wins.** The assistant is asked to make a feature work. A repository that's already on the classpath is the shortest path to a working feature. Nothing in the code says "don't."

**It sees code, not intent.** Your bounded contexts exist in a whiteboard photo, a Confluence page and the heads of three senior engineers. The AI reads `src/main/java`. If a public setter exists, the AI reads that as permission to call it.

**There is no ubiquitous language in the prompt.** "Add birthday points at checkout" puts the rule in checkout. A domain expert would say "Loyalty awards double points for orders paid on the customer's birthday." Same feature, different owner.

This is exactly how a new team behaves when it joins without onboarding. We already know the fix for that team. It works for the AI too.

## Step 1: Ask the AI to discover the domain first

Before writing rules, find out what the code already says. Use the AI as a reader, not a writer.

```text
Read the packages under com.shop.domain.loyalty.
Build a glossary of the domain terms you find: class names, method
names, enum values, event names. For each term give a one-line
definition based only on the code. Mark terms that mean different
things in different places, and terms that look like technical
names rather than business language (e.g. "updateRecord").
Do not suggest changes yet.
```

Then look for the leaks:

```text
List every class outside com.shop.domain.loyalty that imports a
Loyalty entity or repository, or reads/writes the loyalty_account
table through a repository, JdbcTemplate or native query.
Calls to LoyaltyService that only get DTOs back are fine. Skip them.
For each one, show the file, the line, and whether it reads or
writes. Group the results by calling module.
```

The second list is your boundary violations. Controllers and app-layer services that call `LoyaltyService` and get DTOs back are not on it, because that's the intended way in. The list is often longer than expected. Each entry is either a missing public API or a missing event.

## Step 2: Write the context down where the AI will read it

Put an `AGENTS.md` in the Loyalty domain package, `com/shop/domain/loyalty/`. It's a shared convention: Codex, Cursor, GitHub Copilot, Gemini CLI and other assistants read it when they work in that directory. Keep it short and specific.

Recent versions of Claude Code read `AGENTS.md` too, but only when the project has no `CLAUDE.md`. If your repo already has one, Claude skips `AGENTS.md` by default. Don't copy the rules into a second file, because two copies drift apart. Add a `CLAUDE.md` next to `AGENTS.md` that imports it:

```markdown
@AGENTS.md
```

Now every assistant on the team reads the same rules, and you maintain them in one place.

```markdown
# Loyalty context
Purpose: award and redeem points. Loyalty owns the balance. Nobody else writes it.

## Language
- Points: whole, non-negative amount. Never "credits" or "bonus".
- Earn: add points for a paid order. Redeem: spend points at checkout.
- Earning rule: how many points an order is worth (base rate, birthday 2x).

## Invariants
1. balance >= 0, always.
2. Points are earned only for paid orders. Only com.shop.listener calls earnForPaidOrder().
3. Only LoyaltyService changes a LoyaltyAccount. The entity holds data, no logic.
4. Every state change publishes an event in the same transaction.

## Public API
- LoyaltyService: balanceOf(CustomerId), redeem(CustomerId, Points),
  earnForPaidOrder(...) (listeners only). Returns DTOs, never entities.
- Events out: PointsEarned, PointsRedeemed.
- Loyalty doesn't listen to other domains. App-layer listeners call its service.

## Forbidden
- Code outside domain/loyalty using its entity or repository packages.
- Loyalty using other domains' entities or repositories. Use their service or events.
- Writing loyalty tables from SQL, scripts or admin tools.
```

Two things matter here. The language section gives the AI the words to use in names and commits. The forbidden section names the shortcuts you've already seen it take.

A real system has more than one context. Don't put them all in one file. Give each domain package its own `AGENTS.md`, and keep a short context map at the root:

```text
src/main/java/com/shop/
├── AGENTS.md          ← context map: one line per domain + global rules
├── controller/        ← REST controllers
├── service/           ← use cases that combine several domains
├── dto/               ← request/response DTOs for the frontend
├── listener/          ← event listeners that connect domains
├── common/            ← shared kernel: Money, CustomerId, OrderId
└── domain/
    ├── loyalty/
    │   ├── AGENTS.md  ← Loyalty only
    │   ├── service/   ← LoyaltyService: the only entry point
    │   ├── dto/       ← what the service returns, plus events
    │   ├── entity/    ← LoyaltyAccount (hidden)
    │   └── repository/ ← hidden
    ├── ordering/      ← same shape, own AGENTS.md
    └── customer/      ← same shape, own AGENTS.md
```

The top-level packages are a thin application layer. When the frontend needs a balance plus order details on one page, a top-level service calls `LoyaltyService` and `OrderService` and builds the response DTO. When one domain's event should trigger work in another, a listener in `listener/` connects them. Neither layer ever injects a domain's repository.

![Diagram: the application layer in com.shop contains controller/, service/, dto/ and listener/. Arrows point down into the domains in com.shop.domain: loyalty/, ordering/ and customer/. Each domain has public service/ and dto/ packages and hidden entity/ and repository/ packages. ArchUnit enforces that nothing outside a domain touches its entity/ or repository/, and that no domain depends on the application layer.](img/diagram-layers.jpg)

The root file says what each domain owns and how others use it: its service or its events. Global rules go here too: domains never depend on the top-level packages, and a domain's service returns DTOs, never entities. Each domain file covers only its own language, invariants and dependencies.

Domains still need some shared language. `earnForPaidOrder()` takes `Money`, `CustomerId` and `OrderId`, and Ordering uses the same types. These live in `common/`, which DDD calls a shared kernel: a small set of types every domain may use.

Keep it to immutable value records like these, with validation and no business logic. No entities, services or frontend DTOs belong there. Every team depends on it, so a change to `common/` needs every team's agreement.

Say this in the root `AGENTS.md`; without that rule, an assistant will start moving anything that two domains need into `common/`.

This split matters for three reasons. Assistants load a nested file only when they work in that folder, so a change to Checkout doesn't pull in fifteen other contexts. Each context keeps its own vocabulary, so "Account" in Loyalty never gets mixed up with "Account" in Billing. And each team owns its file.

If you use Claude Code and your repo already has a `CLAUDE.md`, repeat the trick from above in every domain: put a one-line `CLAUDE.md` with `@AGENTS.md` next to each domain's `AGENTS.md`.

```text
domain/
├── loyalty/
│   ├── AGENTS.md
│   └── CLAUDE.md      ← contains only: @AGENTS.md
└── ordering/
    ├── AGENTS.md
    └── CLAUDE.md      ← contains only: @AGENTS.md
```

It's tempting to import every domain's `AGENTS.md` from the root `CLAUDE.md` instead. Don't. Claude Code loads the root file and everything it imports at the start of every session, so every context's rules would be loaded all the time. A `CLAUDE.md` inside a domain folder is loaded only when Claude works on files in that folder.

## Step 3: Make the AI work inside the context

Now rewrite the feature request. Point it at the context and ask it to reason about invariants before it writes code.

```text
Read com/shop/domain/loyalty/AGENTS.md first.
Feature: Loyalty awards double points for orders paid on the
customer's birthday.
Before writing code:
1. Say which bounded context owns this rule and why.
2. List every invariant from AGENTS.md this change touches.
3. List the classes you plan to change. Flag any outside
   domain/loyalty.
Wait for my confirmation. Then implement it inside the
LoyaltyService with unit tests for each invariant.
```

The "wait for confirmation" step is where you catch a wrong plan. It costs one message instead of a code review.

The entity only holds data:

```java
@Entity
public class LoyaltyAccount { // com.shop.domain.loyalty.entity
    private CustomerId customerId;
    private MonthDay birthday;
    private Points balance;
    // getters and setters, no logic
}
```

`Points` is a record that rejects negative values in its constructor, so `balance >= 0` holds no matter who sets the balance.

The rules live in `LoyaltyService`. It's the only class that changes a `LoyaltyAccount`, and every change publishes an event in the same transaction:

```java
@Service
public class LoyaltyService { // com.shop.domain.loyalty.service

    private final LoyaltyAccountRepository accounts;
    private final ApplicationEventPublisher events;

    // constructor and balanceOf() omitted

    @Transactional
    public void earnForPaidOrder(CustomerId customerId, OrderId orderId,
                                 Money paidAmount, LocalDate paidOn) {
        LoyaltyAccount account = load(customerId);
        Points earned = Points.forAmount(paidAmount);
        if (MonthDay.from(paidOn).equals(account.getBirthday())) {
            earned = earned.times(2);
        }
        account.setBalance(account.getBalance().plus(earned));
        accounts.save(account);
        events.publishEvent(new PointsEarned(customerId, orderId, earned));
    }

    @Transactional
    public PointsDto redeem(CustomerId customerId, Points points) {
        LoyaltyAccount account = load(customerId);
        if (account.getBalance().isLessThan(points)) {
            throw new InsufficientPointsException(customerId, points, account.getBalance());
        }
        account.setBalance(account.getBalance().minus(points));
        accounts.save(account);
        events.publishEvent(new PointsRedeemed(customerId, points));
        return new PointsDto(account.getBalance());
    }

    private LoyaltyAccount load(CustomerId customerId) {
        return accounts.findByCustomerId(customerId)
                .orElseThrow(() -> new LoyaltyAccountNotFoundException(customerId));
    }
}
```

The setters are public, so in principle anyone could call them. Two things stop that: the entity package is closed to code outside Loyalty, and inside Loyalty only `LoyaltyService` may call the setters. Step 5 turns both into tests.

The balance change and the event are in one transaction. Listeners annotated with `@TransactionalEventListener` run only after that transaction commits, so a rolled-back change never sends an event.

Neither domain knows about the other. Ordering publishes `OrderPaid` without knowing who listens. Loyalty exposes `earnForPaidOrder()` without knowing where paid orders come from. The rule that links them, "a paid order earns points", lives in the application layer, in a listener that combines the two domains. It translates the event into a call and does nothing else:

```java
@Component
class OrderPaidListener { // com.shop.listener

    private final LoyaltyService loyalty;

    OrderPaidListener(LoyaltyService loyalty) {
        this.loyalty = loyalty;
    }

    @TransactionalEventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(OrderPaid event) {
        loyalty.earnForPaidOrder(event.customerId(), event.orderId(),
                event.amount(), event.paidOn());
    }
}
```

Checkout doesn't know Loyalty exists. A failed payment never produces `OrderPaid`, so it never produces points.

![Diagram: OrderService in Ordering commits a successful payment and publishes OrderPaid after commit. OrderPaidListener in com.shop.listener receives it and calls LoyaltyService, which loads the account, doubles the points on the customer's birthday, updates the balance, saves, and publishes PointsEarned to email and analytics. A failed payment rolls back, so there is no OrderPaid and no points.](img/diagram-event-flow.jpg)

`earnForPaidOrder()` has to be public so the listener can call it. That also means a controller could call it and award points without a paid order. Step 5 closes that gap with a test.

## Step 4: Review in a fresh session

Don't ask the session that wrote the code to review it. It has the same blind spots it had five minutes ago. Open a new session, give it the rules and the diff, and nothing else.

```text
You are reviewing a change to the Loyalty bounded context.
Read com/shop/domain/loyalty/AGENTS.md, then run: git diff main...HEAD
For each invariant and each "Forbidden" rule in AGENTS.md,
answer: kept, broken, or not affected. For "broken", quote the
line. Then list any new terms in names or comments that are not
in the Language section. Do not comment on style.
```

A reviewer with a checklist finds more than a reviewer with a vibe. That's true for people and for models.

## Step 5: Enforce the boundary with tests

Everything so far makes a good outcome more likely. None of it makes a bad outcome impossible. The next prompt, the next model or the next tired developer can still call `loyaltyAccountRepository` from checkout.

![Diagram: five steps rising like stairs. Steps 1 to 4 are prompts, which are suggestions: Discover (glossary and leak list), Write it down (AGENTS.md per domain), Plan first (invariants before code), Fresh review (checklist from AGENTS.md). Step 5, Enforce (ArchUnit tests in the build), is the only guarantee. An arrow runs from "a good outcome is more likely" to "a bad outcome fails the build".](img/diagram-guardrails.jpg)

Prompts are suggestions, tests are guarantees. Turn the forbidden section of `AGENTS.md` into ArchUnit rules:

```java
@AnalyzeClasses(packages = "com.shop", importOptions = ImportOption.DoNotIncludeTests.class)
class LoyaltyBoundaryTest {
    @ArchTest
    static final ArchRule internalsStayInside =
            classes().that().resideInAnyPackage("com.shop.domain.loyalty.entity..",
                            "com.shop.domain.loyalty.repository..")
                    .should().onlyBeAccessed().byClassesThat()
                    .resideInAPackage("com.shop.domain.loyalty..")
                    .because("domain/loyalty/AGENTS.md: use LoyaltyService");

    @ArchTest
    static final ArchRule onlyServiceChangesAccount =
            methods().that().areDeclaredIn(LoyaltyAccount.class)
                    .and().haveNameMatching("set[A-Z].*")
                    .should().onlyBeCalled().byClassesThat()
                    .resideInAPackage("com.shop.domain.loyalty.service..")
                    .because("domain/loyalty/AGENTS.md: invariant 3");

    @ArchTest
    static final ArchRule onlyListenersEarnPoints =
            methods().that().areDeclaredIn(LoyaltyService.class)
                    .and().haveName("earnForPaidOrder")
                    .should().onlyBeCalled().byClassesThat()
                    .resideInAPackage("com.shop.listener..")
                    .because("domain/loyalty/AGENTS.md: invariant 2");

    @ArchTest
    static final ArchRule domainsDoNotDependOnAppLayer =
            noClasses().that().resideInAPackage("com.shop.domain..")
                    .should().dependOnClassesThat().resideInAnyPackage(
                            "com.shop.controller..", "com.shop.service..",
                            "com.shop.dto..", "com.shop.listener..")
                    .because("dependencies point one way: app layer -> domain");
}
```

(`classes`, `noClasses` and `methods` are static imports from `ArchRuleDefinition`.)

The `because()` text points back to `AGENTS.md`. When the AI breaks a rule, the failing build tells it which rule and where it's written. Most assistants will read that and fix the code on their own.
## Takeaways

- Treat the AI as a new team. It needs onboarding, not more detailed tickets.
- Use the AI to discover the domain before you ask it to change it. The glossary and the leak list are useful even without AI.
- Keep `AGENTS.md` next to the code it describes: language, invariants, public API, forbidden dependencies.
- Ask for affected invariants before code, and review in a fresh session against the written rules.
- Every rule you care about should end up as a test. Prompts are suggestions, tests are guarantees.
