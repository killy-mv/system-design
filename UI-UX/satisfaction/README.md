# Satisfaction

> **Satisfaction** = how pleasant and comfortable the product is to use, and how users feel about it: their attitudes, emotions and willingness to keep using and recommend it.
> "Do people *like* using this, or merely put up with it?"

Satisfaction is one of the three parts of usability in ISO 9241-11 (effectiveness, efficiency, **satisfaction**) and one of Nielsen's five usability components ([Usability §2](../usability/README.md#2-the-five-components-of-usability)). In ISO/IEC 25010:2023 it relates to **user engagement**. Unlike the other attributes, it's **subjective**: it's measured by asking users, not by timing them.

---

## 1. What drives satisfaction

Satisfaction is mostly the *result* of other qualities, plus some emotional extras:

| Driver | Notes |
|---|---|
| **It works and it's reliable** | Bugs, data loss and outages destroy satisfaction faster than anything else. Reliability is the foundation. |
| **It solves the user's problem** (usefulness) | Useful = utility + usability |
| **It's fast and responsive** | Slowness is one of the most common complaints ([Responsiveness](../responsiveness/README.md)) |
| **It's easy** | Low effort, few errors, easy recovery |
| **It's respectful** | No dark patterns, no nagging, honest pricing, easy cancellation, privacy respected |
| **It feels good** | Visual polish ([Aesthetics](../aesthetics/README.md)), pleasant microinteractions, a friendly voice, moments of delight |
| **Support when things go wrong** | Fast, human, effective help; how failures are handled shapes the memory of the whole experience |

### 1.1 Models worth knowing

- **Norman's three levels of emotional design** (*Emotional Design*, 2004): **visceral** (immediate reaction to how it looks and feels), **behavioral** (the experience of using it: effectiveness, ease), **reflective** (what it means to the user: self-image, memories, story).
- **Kano model**: features fall into **basic needs** (expected; their absence causes dissatisfaction, their presence doesn't delight), **performance needs** (more is better, roughly linear), and **delighters** (unexpected; cause delight, but their absence isn't noticed). Delighters become basics over time (what once delighted, like dark mode or autosave, is now expected).
- **Peak-end rule**: people judge an experience mostly by its most intense moment and its ending. A great checkout followed by a confusing confirmation, or a smooth app with one painful error, are remembered by those moments.
- **Aesthetic-usability effect**: people perceive attractive designs as easier to use and are more tolerant of their minor problems.

**Delight is not a substitute for basics.** Confetti animations on a slow, buggy app make users angrier, not happier.

---

## 2. Measuring satisfaction

| Instrument | What it measures | Notes |
|---|---|---|
| **SUS** (System Usability Scale) | Perceived usability, 10 questions, score 0–100 | Industry standard; ~68 is roughly average ([Usability §9.2](../usability/README.md#92-attitudinal-self-reported-metrics)) |
| **UMUX-Lite** | Perceived usability, 2 questions | Short alternative to SUS |
| **SEQ** (Single Ease Question) | Ease of one task, 1–7 | Asked right after a task in a test or in-product |
| **CSAT** (Customer Satisfaction) | "How satisfied were you with…?" (1–5) | Good for specific interactions (support ticket, delivery) |
| **CES** (Customer Effort Score) | "How easy was it to…?" | Effort is a strong predictor of loyalty in service interactions |
| **NPS** (Net Promoter Score) | "How likely are you to recommend…?" (0–10) | Popular with management; criticized as a usability measure (a single noisy question about intent, influenced by many non-product factors). Use with care, alongside other measures. |
| **Google HEART "Happiness"** | Attitudinal metrics as part of a goals–signals–metrics framework | [Usability §9.4](../usability/README.md#94-googles-heart-framework) |
| **Qualitative feedback** | Open comments, interviews, app store reviews, support tickets, social media | Explains *why* the scores move |
| **Behavioral proxies** | Retention, churn, repeat usage, referral, feature adoption | Behavior often contradicts stated satisfaction; combine both |

Good survey practice:
- Ask **at the right moment** (right after a task, not randomly mid-flow).
- **Sample** users and limit frequency, to avoid survey fatigue.
- Track **trends and segments** (new vs experienced users, plans, platforms), not single numbers.
- Close the loop: tell users what changed because of their feedback.

---

## 3. System-design aspects

- **Reliability and performance are satisfaction features**: SLOs, error budgets and latency budgets protect satisfaction directly.
- **Feedback pipelines**: in-app feedback widgets, survey tools, app store review monitoring, support ticket tagging. Categorize feedback at scale (tags, or LLM-assisted clustering with human review) and link it to product areas.
- **Graceful failure**: friendly error states, preserved user work, status pages and honest incident communication soften the worst moments (the "peaks").
- **Personalization and preferences** (themes, density, notifications) let users shape the product to their taste.
- **Respectful notifications**: frequency caps, easy opt-out; notification spam is a top driver of uninstalls.

---

## 4. Anti-patterns

| Anti-pattern | Why it hurts |
|---|---|
| **Dark patterns** (confirmshaming, hidden costs, hard cancellation, forced continuity) | Boost short-term metrics, destroy trust and satisfaction, and attract regulators ([Usability §16.2](../usability/README.md#162-dark-patterns-deceptive-design)) |
| **Survey spam** | Annoys users and biases results toward the most tolerant ones |
| **Optimizing the score instead of the experience** | Asking for ratings only after happy moments, or gating reviews |
| **Delight on top of broken basics** | Highlights the gap |
| **Ignoring qualitative feedback** | Scores show *that* something is wrong, rarely *what* |

---

## 5. Checklist and further reading

- [ ] Basics first: reliable, fast, useful, easy
- [ ] Satisfaction measured with standard instruments (SUS / SEQ / CSAT / CES) and tracked over time by segment
- [ ] Qualitative feedback collected, categorized and acted on
- [ ] Peaks and endings designed deliberately (errors, onboarding completion, checkout confirmation, cancellation)
- [ ] No dark patterns; notifications respectful
- [ ] Delight added only where basics are solid

**Further reading**: Don Norman, *Emotional Design* (2004); John Brooke, *SUS: A Quick and Dirty Usability Scale* (1996); Jeff Sauro & James Lewis, *Quantifying the User Experience*; Noriaki Kano et al. on the Kano model (1984); Kerry Rodden et al., *Measuring the User Experience on a Large Scale* (Google HEART, 2010); Dixon, Freeman & Toman, *Stop Trying to Delight Your Customers* (HBR, 2010, on customer effort).
