# Modifiability

> **Modifiability** = a change can be made **quickly and safely**, touching **few places**, without introducing defects or degrading quality.
> "Adding a new payment method: is that 1 file and an afternoon, or 15 files and a week of fear?"

In **ISO/IEC 25010**, modifiability is a sub-characteristic of maintainability: the degree to which a system can be modified without introducing defects or degrading existing quality.

Related: [Modularity](../modularity/README.md) (the structure that makes changes local), [Simplicity](../simplicity/README.md), [Testability](../testability/README.md) (what makes changes safe), [Evolvability](../evolvability/README.md) (the long-term, architecture-level version), [Extensibility](../extensibility/README.md) (a special case: adding without modifying).

---

## 1. The cost of a change

A change costs: **finding** where to change + **understanding** the code + **making** the edit + **verifying** nothing broke + **deploying** it. Modifiability reduces all five:

| Cost | Reduced by |
|---|---|
| Finding | Clear structure, package by feature, good names ([Analyzability](../analyzability/README.md)) |
| Understanding | Simplicity, deep modules, small scope |
| Making | Low coupling, no duplication of knowledge, localised decisions |
| Verifying | Fast, trustworthy tests ([Testability](../testability/README.md)) |
| Deploying | CI/CD, feature flags ([Deployability](../../operations-devops/deployability/README.md)) |

---

## 2. Code smells that hurt modifiability

| Smell | What it looks like | Fix |
|---|---|---|
| **Shotgun surgery** | One change requires small edits in many classes | Move the scattered knowledge into one place |
| **Divergent change** | One class changes for many unrelated reasons | Split by reason to change (single responsibility) |
| **Duplicated knowledge** | The same rule copied in 4 places (not just similar code) | One source of truth for each business rule |
| **Switch on type in many places** | `if method == "card" … elif "paypal" …` repeated | Polymorphism / strategy / registry |
| **Feature envy** | Code that mostly uses another module's data | Move behaviour to where the data lives |
| **Primitive obsession** | Money as `float`, IDs as plain strings everywhere | Small value types (`Money`, `OrderId`) |
| **Long parameter lists / flag arguments** | `send(order, True, False, None, 3)` | Parameter objects, separate functions |

### Worked example: adding a payment method

messy-shop handles payment methods with `if/elif` chains in four places: checkout, refunds, receipts, admin reports.

**Before:** adding "bank transfer" means editing **4 files** owned by different parts of the code, plus finding all 4 (someone forgets receipts, so bank transfer receipts show "Unknown").

**After:** one interface and a registry:

```python
class PaymentMethod(Protocol):
    code: str
    def charge(self, order: Order) -> ChargeResult: ...
    def refund(self, charge: Charge, amount: Money) -> RefundResult: ...
    def receipt_label(self) -> str: ...

PAYMENT_METHODS = {m.code: m for m in [CardPayment(), PayPalPayment(), BankTransfer()]}
```

Adding a method = **1 new class + 1 line** in the registry. Checkout, refunds, receipts and reports never change, and the type checker reminds you of every method you must implement. That's shotgun surgery turned into a local change.

---

## 3. Practices

- **Refactor continuously** (small steps, with tests), not in big "cleanup projects". Boy scout rule: leave code a bit better than you found it.
- **Make the change easy, then make the easy change** (Kent Beck): refactor first so the feature becomes simple.
- **One source of truth** per business rule, config value and data item.
- **Small PRs** and trunk-based development: easier to review, test and revert.
- **Feature flags** to change behaviour safely ([Deployability §6](../../operations-devops/deployability/README.md#6-feature-flags)).
- **Measure change effort**: files per PR, lead time for changes, hotspots that change often and break often.

---

## 4. Anti-patterns

- Copy-paste of business rules across modules.
- Fear-driven development: "don't touch that file".
- Big refactoring branches that live for months.
- Changing code without tests, then adding "just one more special case".
- Generalising before the second real use case (that's [simplicity](../simplicity/README.md)'s enemy too).

---

## 5. Checklist

- [ ] A typical feature touches one module and few files
- [ ] Business rules have a single source of truth
- [ ] No repeated `switch`/`if` chains on type across the codebase
- [ ] Refactoring is routine, done in small steps with tests
- [ ] Hotspots (high churn + high complexity) identified and improved

---

## 6. Exercises

1. Look at your last 20 PRs. What's the median number of files changed? Which PR touched the most modules, and why?
2. Find a business rule duplicated in 2+ places in a codebase you know. Consolidate it.
3. Find a type-switch repeated in several places and replace it with polymorphism or a registry.
4. Pick a feature you want to add. First refactor so it becomes a one-place change, then add it. Compare with doing it directly.

---

## 7. Further reading

Martin Fowler, *Refactoring* (2nd ed.): code smells and refactorings; Kent Beck, *Tidy First?*; Michael Feathers, *Working Effectively with Legacy Code*; Adam Tornhill, *Your Code as a Crime Scene* (hotspots); Len Bass et al., *Software Architecture in Practice* (modifiability tactics).
