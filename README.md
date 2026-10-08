# Hardening Engine, JEV variant

### The same engine, with a typed decision layer between the compliance report and the fix

**English** · [Português](README.pt-BR.md) | [Base platform](https://github.com/Alisson-P/hardening-engine)

> A report says what is wrong. A plan says what to fix first, how, and who has
> to be in the room.

This is the public preview of the **JEV variant** of my hardening engine. It is
not a different engine: the baseline, the read-only checker and the bench are
the same. The variant adds one stage between the checker's report and the fix,
a **typed triage** that turns the list of what is out of compliance into a
**remediation plan**, with an order and a route for every finding.

**Start with the main project: [hardening-engine](https://github.com/Alisson-P/hardening-engine)**

That page explains the baseline, the severity rule, the checker and the bench.
This one only covers what the variant adds.

---

## The gap between the report and the fix

The checker says what is out of compliance, and the report already sorts it by
severity. What is left between the report and the change window is a set of
questions the catalog cannot answer on its own, because they depend on the
machine:

- did the reading really measure what the item asks for?
- is the value found as strict as the one required, or stricter, and only the
  literal comparison failed?
- does this host even run the component? Is there a compensating control? An
  exception in force?
- can the fix lock out whoever administers the machine? Does it need a
  restart? Will it be lost on the next boot?

Today a person answers that, finding by finding, and the answer is rarely
written down. The variant asks the same questions to a decision model, one at
a time, and lets the code decide, with thresholds written in advance.

---

## Two layers in the plan

**The base plan needs no model at all.** It comes from the catalog alone:

| Field | Rule |
|---|---|
| Priority | critical P1, high P2, medium P3, low P4 |
| Route | image script when the item is automatable, verifiable and its command passed validation; otherwise a maintenance window |
| Access lock | a fix that touches remote access, authentication, privilege elevation or the firewall goes to the service owner, by fixed rule |

**The typed layer sits on top, and it can only do three things:**

1. **lower** a priority by one step, and only with high confidence
2. **raise** the route up the ladder, never down
3. **add signals** for whoever is going to apply the fix

The ladder, from the least to the most human:

```
image script  <  maintenance window  <  service owner  <  exception review
```

With no model available, every question becomes an abstention and the plan
comes out **identical to the base**. The layer is never a requirement for the
cycle to run.

---

## The questions

The model answers in three typed forms, never in prose:

| Primitive | Question | Returns |
|---|---|---|
| Noul | Is this statement true? | a probability between 0 and 1 |
| Choice | Which of these options? | the option, the whole distribution and the confidence |
| Score | Which level on this rubric? | the level, the whole distribution and the confidence |

Each finding gets **eleven questions in a single request**, and each compliant
item gets one more of its own:

| Group | What it asks | Primitive |
|---|---|---|
| Was the finding right? | whether the reading fails to measure what the item requires; whether the value found is equivalent or stricter | Noul |
| Does it apply here? | whether the component is outside the host's role; whether a listed compensating control covers it; whether an exception in force covers it | Noul |
| What does the fix do? | whether it can lock out administrative access; whether it needs a restart; whether it is lost on the next boot | Noul |
| How much is at stake? | the risk of breaking the host's function; the exposure of the component | Score, 4 levels |
| Who should apply it? | the suggested route, the four rungs plus "other" | Choice, 5 options |
| Compliant items | whether the output only shows that something exists, without proving the value | Noul |

The state sent with the questions carries **facts only**: the item, the reading
and what it returned, and the host's role, services, access, exposure,
controls and exceptions in force. **No severity**, so the model does not anchor
on a grade the project already gave. No machine name, no address, no name of
who approved an exception. The reading output goes through a sanitizer that
cuts it at 400 characters and masks password hashes, IP addresses and e-mail
addresses.

---

## Where the decision lives

The model answers questions. The **code** composes the answers, with weights
and thresholds written down, and each threshold follows the cost of the error.

**Lowering a true finding is the expensive mistake:** the weakness stays open
for longer. So lowering requires a confidence of 0.70, and never more than one
step.

**Sending a fix to its owner when it did not need to is the cheap mistake:** it
costs a conversation. So the caution signals act at a probability of 0.50,
without asking for confidence.

**Exception review only comes from the context group, with high confidence.**
It is the most human rung, and it also means "do not fix it now", which is the
expensive mistake again. If the suggested route asks for an exception without
the context backing it, the line gets a signal and the route stays where it
was.

Confidence is computed in the code, with the same formulas for every model, so
a threshold means the same thing whichever model answers. The contract and the
formulas follow the official Jev documentation. One detail mattered more than
it looks: a Score's confidence is centred on the **most likely** level.
Centring on the mean or on the median looks equivalent and is not. For a
rubric answered as (0, 0.40, 0.25, 0.35, 0), the most likely level gives a
confidence of 0.208, the mean 0.367 and the median 0.375.

---

## The locks

What no answer from the model gets past:

1. a critical item never loses priority
2. no finding drops more than one step
3. the route never goes down the ladder
4. no finding leaves the plan
5. an expired exception never reaches the model: the code filters it by date
   first
6. for a hosted service, the state leaves redacted or does not leave
7. an answer outside the contract becomes an abstention, never an invented
   value

At the end, the triage checks the whole plan against the base, and any
violation stops everything: the plan is not written. Then a separate reader
compares the two plans again, from the files alone, without importing anything
from the triage.

The triage never writes to the machine. It reads two files and writes the plan.

---

## Where the model runs

| Option | What it is | Does the state leave the environment? |
|---|---|---|
| Offline | no model, every answer is an abstention, plan identical to the base | no |
| Local decision model | an open decision model served on the machine itself, the default | no |
| Local language model | an Ollama compatible model, probability by self consistency | no |
| Hosted Jev API | the hosted decision service, kept as a comparison referee | yes, redacted only |

The local decision model and the hosted API speak **the same contract**, so one
replaces the other without changing a line of the triage. The quality of the
open model in this domain has **not been measured** yet, and the repository
carries a validator that has to pass before it is trusted.

---

## How it is proven

**A known-answer control, 40 of 40.** Every rule of the composition has a case
with the right answer written before it runs, including the worked examples of
the Jev documentation itself.

**Rules broken on purpose, 25 of 25 caught.** Each one damages a single thing
(lowering two steps, letting the route go down, centring the Score on the
mean) and the control has to fail. A control that never fails proves nothing.

**End to end on a real Ubuntu machine.** A throwaway GitHub runner: the checker
reads it, the plan without a model comes out equal to the base, the plan with a
simulated service comes out marked as simulated on every line of the audit
trail, and the locks are checked again from outside. Green in under a minute.

---

## The benchmark

**Read this before the numbers.** Model inference time was **not measured**.
The answers came from a simulated service that speaks the same contract and
answers by rule. So these numbers measure the cost of the contract: assembling
the state, transport, validating the answer, composing in code, locks and
audit trail. The model's own time sits on top of it, and it depends on
hardware.

The input reports are synthetic. The item, the reading, the criterion and the
detail come from the catalog and from the engine's own comparison; only the
value read is invented. Four host profiles, with 100, 1,000 and 5,000 findings
each, on a one core Linux machine. One request per item, ranges across the four
profiles:

| Findings | Average per request | p95 | Findings per second | Locks violated |
|---:|---:|---:|---:|---:|
| 100 | 2.6 to 4.4 ms | 2.7 to 7.5 ms | 143 to 236 | 0 |
| 1,000 | 2.7 to 2.8 ms | 2.9 to 3.1 ms | 218 to 232 | 0 |
| 5,000 | 2.7 to 2.9 ms | 2.7 to 3.9 ms | 216 to 229 | 0 |

The cost is per request and does not grow with the size of the report. At 100
findings the first calls pay for the service warming up. Across the twelve runs,
**24,400 findings** were triaged, no lock was violated, and the same input
produced the same plan. On the GitHub runner, with 200 findings per profile, the
contract cost about 1.5 to 1.6 ms per request.

**What the layer changes depends on the host.** With 1,000 findings:

- the graphical workstation gets fewer "context attenuates" signals, because
  login screen items are not outside its role there
- the web server gets more "requires restart" signals, because the fixes that
  restart the logging service hit a service it provides
- the Kubernetes node gets more "high break risk" signals and more routes
  raised, 166 against 145, because turning packet forwarding off breaks a node
  that forwards traffic between pods

**Routes only went up**, in every run. **No exception candidate appeared**: with
simulated answers, the context group never gathered enough value and confidence
at the same time. The path exists and the control proves it, but this corpus
did not exercise it. The other side shows in the "model suggests exception"
signal: the suggested route asked for an exception hundreds of times, and the
code did not let it become a route without the context backing it.

> This measures what the layer costs and what it is allowed to change, not
> whether it changes it correctly. Turning that into evidence needs a real
> model against a hand labelled sample.

---

## Which of the two to use

**The base, when a person reads the report.** The base plan already orders by
severity and protects access by rule. With a few dozen findings and an
administrator who knows the hosts, the variant adds little.

**The variant, when the fleet passes what anyone can read.** A typed answer
with calibrated confidence can be applied by rule: above the threshold the plan
adjusts on its own, below it nothing changes and the finding stays where the
base put it. That only works because the confidence is a comparable number.

## What the variant costs

**A model with typed output.** The open model still has to be measured in this
domain, and that is the first thing to validate where it will run.

**One more request per item.** Milliseconds for the contract, plus the model's
inference on top.

**More to maintain.** Questions, rubrics, weights and thresholds live in the
repository, and they age.

## Why the base is still the main project

The engine works **with no artificial intelligence at all**: the baseline, the
checker, the bench and the base plan do not depend on a model. Switch the layer
off and a complete plan remains. The model refines the order and the route, it
does not hold the plan up, and it never touches the machine.

---

## What this preview shows, and what it does not

**It shows:** the design of the triage, the reasoning behind each threshold, the
locks and the numbers.

**It does not show:** the code, the question bank, the weights, the decision
service and the full benchmark document. All of that lives in a private
repository.

No data from a real environment, a client or an assessed machine appears in
this preview. The benchmark reports are synthetic, and the end to end run used a
throwaway machine.

If you would like to see the full content, get in touch.

---

## About

I am Alisson Pereira, I work with cloud security. The base engine tells what is
wrong without ever writing to the machine. This variant keeps that intact and
adds one decision stage, where the model answers small questions and the code
decides, so the order and the route of every fix can be audited answer by
answer.

[LinkedIn](https://www.linkedin.com/in/alisson--pereira/) · [Other projects](https://github.com/Alisson-P)

---

## License

This preview is licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/).

You may share it with credit. You may not use it commercially nor distribute a
modified version.
