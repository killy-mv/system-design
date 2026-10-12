# Analyzability

> **Analyzability** (often called **readability** or **understandability**) = how easily people can **understand** the system: what it does, why, how a change will affect it, and where a problem comes from.
> "Can someone who didn't write this code figure out what it does and what will break if they change it?"

In **ISO/IEC 25010**, analysability is a sub-characteristic of maintainability: assessing the impact of a change, diagnosing deficiencies and causes of failure, and identifying parts to modify.

Related: [Simplicity](../simplicity/README.md) (less to understand), [Modularity](../modularity/README.md) (understand one part at a time), [Debuggability](../../operations-devops/debuggability/README.md) (diagnosing problems at runtime), [Maintainability §12](../maintainability/README.md#12-documentation-and-knowledge-sharing) (documentation).

---

## 1. Why it matters

- Code is **read far more often than it's written**: in reviews, debugging, onboarding, and before every change.
- You can't safely change what you don't understand, so analyzability limits [modifiability](../modifiability/README.md).
- It's what lets **impact analysis** work: "if I change this, what else is affected?"

---

## 2. What makes code easy to understand

| Level | Practices |
|---|---|
| **Names** | Names that say what and why (`unpaid_invoices_older_than_30_days`, not `list2`); domain language from the business |
| **Functions** | Small, one level of abstraction, few parameters, no hidden side effects; guard clauses instead of deep nesting |
| **Types** | Types that document intent (`Money`, `OrderStatus`, `EmailAddress`); illegal states unrepresentable |
| **Structure** | Package by feature; consistent layout across modules; obvious entry points |
| **Comments** | Explain **why** (decisions, constraints, links to tickets/ADRs), not what the code already says |
| **Consistency** | One style (formatter + linter), one way to handle errors, logging, config |
| **Architecture docs** | C4 diagrams (context, containers, components), ADRs for decisions, a README per module |
| **Runtime understanding** | Logs, traces and metrics that show what the system actually does ([Observability](../../operations-devops/observability/README.md)) |

### Worked example: cognitive complexity

SonarSource's **cognitive complexity** metric adds +1 for each `if`, loop, `catch` etc., plus **+1 more for each level of nesting** it's inside. It models how hard code is to follow.

```python
# Before: nested                                              increments
def ship_ready_orders(orders, warehouse):
    if warehouse.is_open:                                     # +1
        if warehouse.has_staff:                               # +2 (nesting 1)
            for order in orders:                              # +3 (nesting 2)
                if order.is_paid and not order.is_shipped:    # +4 (nesting 3) +1 (&& sequence)
                    warehouse.ship(order)
# Cognitive complexity = 11
```

```python
# After: guard clauses + a named predicate
def ship_ready_orders(orders, warehouse):
    if not warehouse.is_open:                                 # +1
        return
    if not warehouse.has_staff:                               # +1
        return
    for order in orders:                                      # +1
        if order.ready_to_ship():                             # +2 (nesting 1)
            warehouse.ship(order)
# Cognitive complexity = 5
```

Same behaviour, about half the mental effort, and `ready_to_ship()` now has a name you can test and reuse. Use the metric to **find** the worst functions, not as a target to game.

---

## 3. Finding where to focus: hotspots

Not all code needs to be equally clear. Focus on **hotspots**: code that is both **complex** and **frequently changed** (from Git history). A messy file nobody touches costs little; a messy file changed every week costs a lot. Tools: CodeScene, code-maat, or a script combining `git log` churn with a complexity tool (radon, lizard, SonarQube).

---

## 4. Anti-patterns

- Abbreviations and generic names (`data`, `info`, `manager`, `handle`, `tmp2`).
- Deep nesting and 300-line functions.
- Comments that repeat the code, or are outdated and wrong.
- Clever one-liners and implicit magic.
- Several styles and conventions in one codebase.
- Architecture knowledge only in people's heads.
- No way to see what the system actually does at runtime.

---

## 5. Checklist

- [ ] Formatter and linter enforced in CI
- [ ] Complexity checked; worst functions in hotspots refactored
- [ ] Domain language used consistently in code
- [ ] Comments explain why; ADRs for decisions
- [ ] C4-style diagrams and a README per module, kept up to date
- [ ] New engineers can find where a feature lives within minutes

---

## 6. Exercises

1. Compute the cognitive complexity of the most complex function you wrote recently. Refactor it below 10.
2. Rename 10 things in a codebase you know to use the business's words. Did any reveal a misunderstanding?
3. Draw a C4 context and container diagram for one of your lab systems. Ask someone to answer "where is X handled?" using only the diagram.
4. Pick an unfamiliar open-source repo. Time how long it takes to find where a specific feature is implemented. Write down what helped and what got in the way.

---

## 7. Further reading

Dustin Boswell & Trevor Foucher, *The Art of Readable Code*; Robert C. Martin, *Clean Code* (read critically); John Ousterhout, *A Philosophy of Software Design* (comments, obscurity); SonarSource, "Cognitive Complexity" white paper (G. Ann Campbell); Simon Brown, the C4 model (c4model.com); Felienne Hermans, *The Programmer's Brain*.
