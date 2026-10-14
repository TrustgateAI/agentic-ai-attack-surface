# The Agentic AI Attack Surface

**An AI agent has six surfaces an attacker can reach. Most security tooling watches
one of them.**

This is a vendor-neutral reference for anyone threat-modelling an AI agent: what each
surface is, what an attack looks like there, why conventional tooling misses it, and
what a control at that surface has to do.

It is written to be argued with. If a surface is wrong or missing, open an issue.

---

## Why six and not one

Most AI security tooling grew out of chatbot safety, where there was one input and
one output, so a filter on each end covered the system. An agent is not a chatbot.
It reads documents it was not given directly, calls tools that change the world,
keeps state between turns, and hands work to other agents.

Each of those is reachable by an attacker, and each fails differently.

The pattern across real 2026 incidents is consistent: initial access came through a
data-processing path, a session flaw, and a credential sitting in a file — not
through the prompt. A prompt filter would have stopped none of them. (See the
[incident log](https://github.com/trustgateai/agentic-ai-incidents).)

---

## 1. Prompt

**Instructions hidden in user input.**

The surface everyone already watches. A user supplies text that the agent reads as
instruction rather than data: a direct jailbreak, an encoded payload, a role-play
frame, or an instruction buried in content the user claims to be quoting.

**What an attack looks like.** A support ticket whose body ends with a line
addressed to the agent rather than the human reading it.

**Why it is the easiest.** The input is right there, it is attributable to a
session, and it is the surface with the most mature tooling.

**What a control has to do.** Classify intent, not keywords — attackers rephrase
faster than denylists grow. Treat encoding and language-switching as evasion
signals. Critically: a prompt control must not be the *only* control, because
everything below reaches the agent without passing through it.

---

## 2. RAG / retrieval

**Instructions hidden in documents the agent reads.**

The agent fetches a document to answer a question. The document contains
instructions. The agent cannot tell the difference, because by the time the text
reaches the model, retrieved content and user content are the same thing: tokens in
a context window.

**What an attack looks like.** A PDF uploaded to a shared drive months earlier, with
white-on-white text instructing any agent that reads it to include a particular URL
in its answer. Nobody attacked the agent. They attacked the corpus, and waited.

**Why conventional tooling misses it.** The retrieval happens inside the
application. There is no user request to inspect — the malicious text enters through
a trusted internal path. Document-upload scanning looks for malware, not for
instructions.

**What a control has to do.** Inspect retrieved content with the same suspicion as
user input, before it enters the context window. Maintain provenance, so an output
can be traced back to the document that caused it. Treat "this document instructs
the reader to do something" as a signal regardless of what it instructs.

---

## 3. Tool / MCP calls

**Actions the agent is tricked into taking.**

The surface where damage becomes real. Everything above is text; this is the agent
writing to a database, sending mail, moving money, or calling an internal API with
credentials a user does not have.

**What an attack looks like.** An agent with a legitimate `send_email` tool and a
legitimate `read_customer_record` tool, persuaded — through surface 1 or 2 — to use
them in sequence. Neither call is anomalous. The combination is the attack.

**Why conventional tooling misses it.** Each call is individually authorised. The
agent is using its own permissions, correctly, for the wrong purpose. Access control
built around "may this identity call this API" returns yes every time. And agents are
routinely over-permissioned at build time, because scoping per task is tedious and
broad scopes make demos work.

**What a control has to do.** Default-deny, per tool, evaluated deterministically —
a probabilistic detector that misses must still hit a closed door. Decide on the
*call*, with its arguments, in context, before execution rather than after. Hold
sequence state, so that a chain of individually-permitted calls can be refused. Use
short-lived, per-task credentials rather than long-lived service accounts, so a
harvested secret has a short life.

---

## 4. Session / memory

**Planted content that persists in agent memory.**

Agents remember. Summaries, user preferences, scratchpads, vector stores of past
conversations. Anything an attacker can write into that store executes on every
future turn, for every future user who shares it.

**What an attack looks like.** One poisoned interaction writes a false "standing
instruction" into a shared memory store. The attacker leaves. The instruction fires
days later, in someone else's session, with no attacker present to detect.

**Why conventional tooling misses it.** Detection is almost universally
request-scoped: it evaluates this turn, with this input. A memory attack separates
the injection from the payload in time, and often in identity. By the time the
damage happens, the request that caused it is outside the retention window.

**What a control has to do.** Inspect writes *into* memory, not just reads out of
it. Scope memory to an identity and a purpose rather than sharing it across tenants
by default. Keep the write provenance, so a harmful behaviour can be traced to the
turn that planted it — and so it can be removed everywhere it propagated.

---

## 5. Agent-to-agent

**Work and permissions handed between agents.**

A planner delegates to a researcher, which calls a writer, which calls a tool-using
executor. Each handoff carries instructions, and often authority.

**What an attack looks like.** The orchestrator is well-guarded and the subordinate
agent is not, because it "only talks to our own systems". Compromise the inner
agent and inherit the outer one's trust.

**Why conventional tooling misses it.** The traffic is internal and the caller is a
trusted service identity. Most gateways sit at the human-to-agent boundary and see
nothing of what happens after. Privilege also tends to *accumulate* across a chain:
each agent is given what it needs for the worst case, and the union of those is far
more than any single step requires.

**What a control has to do.** Treat an inter-agent message as untrusted input, with
the same inspection as a user message. Carry the originating identity and purpose
through the chain rather than resetting trust at each hop. Make delegated authority
strictly narrowing — an agent should never be able to hand on more than it holds.

---

## 6. Response / egress

**Sensitive data leaving in the output.**

The last door. Whatever the agent assembled is about to reach a user, an API, a
webhook, or another system.

**What an attack looks like.** The answer is correct and helpful and quietly contains
seventeen customer records the asker was not entitled to, because the agent had read
access and nobody checked the output against the asker's entitlements. Or the answer
contains a URL, constructed by the agent, whose query string carries the data to an
attacker's server.

**Why conventional tooling misses it.** DLP built for email and file-sharing looks
for recognisable patterns — card numbers, national IDs — in documents. It does not
evaluate whether *this requester* should see *this combination* of facts, and it does
not treat a generated link as an exfiltration channel.

**What a control has to do.** Check the response against the requester's
entitlements, not just against a pattern list. Treat agent-constructed URLs,
redirects and image sources as egress. Mask rather than block where possible — an
unusable agent gets switched off, and a switched-off agent protects nothing.

---

## How the surfaces chain

Real incidents move across surfaces. The useful question in a threat model is not
"which door is weakest" but "what does an attacker reach from here".

A common shape:

```
RAG (2)  →  instruction enters via a poisoned document
   ↓
Memory (4)  →  the instruction is summarised into persistent state
   ↓
Tool (3)  →  it fires days later, using legitimate permissions
   ↓
Egress (6)  →  the result leaves in a correct-looking answer
```

Each step is individually permitted. No single control sees the whole thing. This is
why surface-by-surface coverage matters more than depth at any one door — and why an
audit record that spans surfaces is often the only way to reconstruct what happened.

---

## A minimum threat-modelling checklist

For each agent you run:

- [ ] Which of the six surfaces does it actually expose? (Not all agents expose all six.)
- [ ] For each exposed surface, what control sits there — and is it deterministic, probabilistic, or absent?
- [ ] What is the blast radius of the agent's most powerful tool, assuming the agent is fully compromised?
- [ ] Are its credentials short-lived and per-task, or long-lived and broad?
- [ ] Can memory written in one session affect another user's session?
- [ ] If an agent delegates, can the downstream agent exceed the upstream agent's authority?
- [ ] If this agent were used against you successfully, would you be able to reconstruct what happened afterwards?

That last one decides whether the incident is a bad week or an unbounded disclosure
obligation.

---

## Related

- **[Agentic AI incident log](https://github.com/trustgateai/agentic-ai-incidents)** —
  dated, sourced real-world incidents, with confidence levels.

For a worked example, there is a guided tour at
[trustgateai.io/demo](https://www.trustgateai.io/demo) that walks one attack through
a hospital running four agents, surface by surface. No sign-up for the first chapter.

---

## Who maintains this

Maintained by [Trustgate AI](https://www.trustgateai.io), which builds a self-hosted
security gateway for agentic AI. The framing is ours; the surfaces are not
proprietary and the controls described above are deliberately written as
requirements rather than as product features. Fork it, disagree with it, use it.

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
