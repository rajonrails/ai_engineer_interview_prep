# C13 — Scoping Under Ambiguity: The Core FDE Skill

> **Day 4, Morning** · Track: FDE · ⭐⭐⭐⭐
> **Read time:** ~20 min. **Why now:** every other topic in this track is knowledge you can
> read. This one is judgment, it's the actual job, and it's the round most strong engineers
> fail. Expect a live roleplay: the interviewer plays a customer with a vague, wrong, or
> self-contradicting request.

---

## The One Idea

**The customer's stated problem is a hypothesis, not a spec.** Your job is not to build what
they asked for — it's to find the real constraint, propose the smallest thing that tests it,
and be honest about what AI will and won't do.

The failure mode on both sides:
- **Order-taker:** builds exactly what was asked, ships in four months, nobody uses it because
  the stated problem wasn't the real one.
- **Know-it-all:** decides in minute three that the customer is wrong, lectures them, loses the
  room and the information they had.

The job is the narrow path between: take the request seriously enough to understand it,
skeptically enough to test it.

---

## 1. The questions that actually move a scoping call

Not a checklist to recite — a set of things you must come away knowing.

**"Walk me through the last time this went wrong."**
Concrete beats abstract every time. "Our support is slow" tells you nothing. A specific ticket
that took three days because an agent couldn't find a policy document tells you it's a
retrieval problem with a specific corpus.

**"Who does this today, and what do they actually do?"**
You cannot automate a process nobody can describe. This also surfaces the tacit knowledge that
will make or break the system — and sometimes reveals the humans are doing something smarter
than the spec admits.

**"What happens if the AI gets it wrong?"**
Determines the entire architecture. Wrong answer costs a re-ask → ship fast, iterate. Wrong
answer costs a regulatory filing → human in the loop, full audit trail, maybe don't use an LLM
for the decision at all. **Ask this early; it constrains more than any other answer.**

**"How will we know it worked?"**
If the answer is "it'll feel better," you don't have a project, you have a vibe. Push until
there's a number, or a set of examples where everyone agrees on the right output. No agreed
success criterion → no project. (Same non-negotiable as the eval-set requirement in C09.)

**"What data exists, and can I see ten examples right now?"**
Not a schema. Ten real rows. The gap between the described data and the actual data is where
timelines die. This one question has killed more bad projects than any architecture review.

**"Who owns this in six months?"**
Names, not teams. If nobody can be named, you're building a demo.

---

## 2. Finding the real constraint

Stated goals are rarely the binding constraint. Listen for which of these is actually true:

| They say | The real constraint might be |
|---|---|
| "It needs to be more accurate" | They have no eval set and accuracy is a proxy for trust |
| "It's too slow" | Perceived latency — streaming or a progress indicator may be the whole fix (see C01 on TTFT vs TPOT) |
| "It's too expensive" | Unit economics don't close at *any* achievable price, and the use case is wrong |
| "We need it in the EU" | Data residency, which eliminates most of your architecture options |
| "We want to fine-tune" | An engineer on their side already decided, and there is internal politics attached (see C09) |
| "Can it do X too?" | Scope creep, or the original scope was never the real goal |

**The politics one is real and worth naming out loud in an interview.** Sometimes a request
exists because a VP promised something. Pretending that isn't a constraint doesn't make it
go away; you handle it by finding a version of the work that is both technically sound and
lets them keep their commitment.

---

## 3. Scope to a two-week test, not a six-month build

Structure the first engagement to **kill the riskiest assumption fastest**:

1. **Name the riskiest assumption.** Usually one of: the data is good enough, retrieval can
   find the answer, the model can do the reasoning, or users will actually adopt it.
2. **Design the cheapest test of it.** Often a notebook and 100 hand-labeled examples. If
   recall@10 over their real corpus is 45%, you learned in three days what six months would
   have taught you.
3. **Agree the decision rule before you start.** "If we hit 80% on these 100 examples, we
   build. If not, we stop or pivot." Writing this down in advance is what makes it a test
   rather than a demo.
4. **Then build**, with the eval set you just created as your regression suite.

**The line worth being able to say:** *"I'd rather find out in two weeks that this doesn't
work than in six months."* Customers respect it, and it is the single most senior-sounding
thing in a scoping conversation.

---

## 4. The demo trap

A demo is 10% of the work and 90% of the perceived progress. This gap destroys FDE
engagements, so manage the expectation explicitly and early.

What's missing between demo and production: evals and regression testing, error handling and
fallbacks, latency under real load, cost at real volume, auth and tenant isolation, PII
handling, monitoring and alerting (C11), prompt and model versioning, rollback, and the long
tail of inputs nobody demoed.

**Say it in the first week, not the twelfth.** "This demo took two days. Production takes
eight weeks, and here's specifically what those eight weeks are." That conversation is
survivable in week one and not in week twelve.

---

## 5. Saying no, usefully

Sometimes the answer is that this shouldn't be built. Refusing well is a senior skill:

- **Never a flat no.** "That specific approach will fail because X. Here's what would work
  instead / here's the cheap test that would tell us."
- **Make the trade-off theirs.** "We can have it in two weeks at 80%, or eight weeks at 92%.
  Given a wrong answer costs a re-ask, I'd take the two weeks — but it's your call."
- **Write down the assumptions.** When requirements shift later, a written assumption list
  turns a conflict into a reference.
- **Escalate scope changes as decisions, not complaints.** "This adds three weeks. Do we
  extend, or drop feature Y?"

---

## 6. How the round is actually run

The interviewer roleplays a customer and is usually deliberately vague or wrong. What they're
scoring:

- Do you **ask before proposing**? Jumping to architecture in minute two is the most common
  failure.
- Do you get to the **real constraint**, or accept the stated one?
- Do you **quantify**? "How many documents? How many users? What's the latency budget?"
- Do you **name what you don't know** and how you'd find out?
- Do you **push back** when the request is wrong, without being condescending?
- Do you propose something **small and testable** rather than a grand architecture?

**Strong signal:** "Before I propose anything, can I ask five questions?" — then asking five
good ones. **Weak signal:** a beautiful system diagram for a problem you haven't confirmed
exists.

---

## Self-check

1. Customer: "we want an AI that answers all our customer questions." Your first five questions.
2. Why does "what happens if it's wrong?" constrain the architecture more than accuracy targets?
3. Design a two-week test for "can AI triage our support tickets?" including the decision rule.
4. A VP promised the board an AI feature that you think is a bad idea. How do you handle it?
5. Your demo wowed them. Script the expectation-setting conversation for the next eight weeks.
