---
layout: post
title: "🕵️ Compliance Academy: A Multi-Agent Cyber Mystery on Microsoft Foundry"
date: 2026-06-10
author: Lee Whieldon
description: How I built a four-agent compliance training RPG for Microsoft Reactor's Reasoning Agents Live Streaming Battle, grounded in real policy retrieval through Azure AI Search.
---

## 🎮 When Compliance Training Meets a Whodunit

When Microsoft Reactor invited me to compete in the [Reasoning Agents Live Streaming Battle](https://developer.microsoft.com/en-us/reactor/), three of us individually had a few minutes each to show what reasoning agents could do live on stream. I had been wanting to do something other than yet another "summarize this PDF" demo, so I went the other direction entirely: **a cyber-mystery role-playing game where compliance training is the prize**.

The result is **Compliance Academy**, a four-agent RPG built on **Microsoft Foundry Agent Service**. You play the lead investigator at *Helix Dynamics*, a fictional biotech that just lost 14 GB of clinical trial data. Five suspects. One perpetrator. Multiple frameworks (SOC 2, HIPAA, ISO 27001) and Helix's own policies are in play. The agents are doing real work the whole time: retrieving from a compliance knowledge index, streaming responses, swapping personas, and at the end the **Compliance Officer** delivers the framework lesson the player just lived through.

The full source is here: 👉 [**github.com/lwhieldon/msft-enterprise-learning-agent**](https://github.com/lwhieldon/msft-enterprise-learning-agent)

---

## 🚀 Why Build a Game and Not Another Q&A Bot?

Most compliance training is something we click through to make a deadline. The information is dense, the stakes are abstract, and the answers are usually "it depends, ask Legal." The result is people learn just enough to pass the quiz, which is exactly the wrong outcome.

A game changes the contract. Suddenly the player has agency. They're asking questions because they need an answer to make a decision, not because the LMS says they have to. They're talking to suspects who are evasive on purpose. They're piecing evidence together. When the Compliance Officer steps in at the end, the framework citation lands because the player just spent twenty minutes earning it.

That was the bet I wanted to make on stream: **reasoning agents are good enough now that we can make compliance training feel like a case file instead of a quiz**.

---

## 🧠 The Architecture

Compliance Academy uses a **Connected Agents** pattern on Microsoft Foundry Agent Service. Four party agents (always present, broad expertise) work alongside a roster of suspect agents (dynamically activated, narrow personas). The player is the orchestrator — they decide who to talk to and when.

| Role | Purpose | Backing |
| --- | --- | --- |
| 🧭 **Game Master** | Scene-setting, action menu, scene close | gpt-4.1-mini |
| 🔬 **Forensic Analyst** | Evidence reasoning, log analysis, framework lookups | gpt-4.1-mini + Foundry IQ retrieval |
| ⚖️ **Compliance Officer** | Post-scene verdict and framework grounding | gpt-4.1-mini + Foundry IQ retrieval |
| 🆕 **Scenario Generator** | Hot-loads brand-new cases from a one-sentence breach prompt | gpt-4.1-mini |
| 👤 **5 Suspects** | Distinct personas — backstory, alibi, leak conditions, hidden truth | gpt-4.1-mini per persona |

The two surfaces matter as much as the agents:

- **Chainlit UI** — what the player and the audience see. Branded, conversational, click-through action buttons.
- **Live activity log terminal** — what proves the orchestration is real. Every Foundry IQ retrieval, every Azure OpenAI POST, every first-token latency, every source name and relevance score streams by in real time.

The activity log is the show-don't-tell part. The Chainlit UI could be a slick wrapper around a single prompt, and nobody in the audience would know. The terminal is how I demonstrate the agents are actually doing work.

---

## 🔎 Grounding the Agents in Real Policy

The thing I care most about with compliance content is **hallucination**. If the Forensic Analyst tells you SOC 2 CC6.1 says something it doesn't, the whole training is worse than useless. So both the Forensic Analyst and the Compliance Officer ground their responses against an **Azure AI Search** index containing 52 chunked policy documents covering SOC 2, HIPAA, ISO 27001, NIST 800-53, and the fictional Helix Dynamics internal policy library.

The retrieval call itself is unremarkable Python:

```python
def retrieve_context(query: str, top_k: int = 5) -> list[dict]:
    """Search the compliance knowledge index and return top-k snippets."""
    client = build_search_client()
    results = client.search(
        search_text=query.strip(),
        top=top_k,
        select=["uid", "snippet", "blob_url", "snippet_parent_id"],
    )
    return [
        {
            "source_url": r["blob_url"],
            "snippet": r["snippet"],
            "score": r["@search.score"],
        }
        for r in results
    ]
```

What makes it land in the demo is the per-source event logging. Each retrieval emits its filename and relevance score to the activity log:

```
[Foundry IQ]    Retrieved 5 sources in 1233ms
[Foundry IQ]      vendor_breach_response.md  (score=11.20)
[Foundry IQ]      helix_dynamics_overview.md  (score=7.65)
[Foundry IQ]      credential_compromise_response.md  (score=7.41)
[Foundry IQ]      credential_compromise_response.md  (score=7.37)
[Foundry IQ]      vendor_breach_response.md  (score=6.93)
[Azure OpenAI]  POST gpt-4.1-mini  (max_tokens=1500, temp=0.4)
[Azure OpenAI]  First token in 8328ms
[Azure OpenAI]  Stream complete: ~816 tokens in 14.9s
```

When the Forensic Analyst then says *"per HD-SEC-AC-001 §4.1, MFA exceptions must be documented in the exception register..."* the audience can see the specific document that the citation came from. The trust loop closes on screen.

---

## 🎭 Suspects as Persona-Driven Agents

Each suspect is its own agent with a templated system prompt that gets filled from scenario data:

```python
SUSPECT_TEMPLATE_VARS = [
    "{{name}}", "{{role}}", "{{premise}}", "{{backstory}}",
    "{{open_knowledge}}", "{{guarded_knowledge}}",
    "{{hidden_truth}}", "{{alibi}}",
    "{{conversational_style}}", "{{style_examples}}",
    "{{leak_conditions}}", "{{starting_trust}}",
]
```

The interesting design choice was **leak_conditions**. Each suspect has a list of triggers like *"when asked directly about the Sofia office"* or *"when pressed twice on the same alibi detail."* When the model sees those triggers in conversation, it's authorized to start leaking guarded knowledge — but only what's in the `guarded_knowledge` block, never the `hidden_truth` block (unless the suspect is the perpetrator and the player has accumulated enough trust).

The result is suspects who *feel* like real people in an interview: defensive at first, slowly opening up, with one of them genuinely hiding something.

---

## 🛠️ The Scenario Generator: Live World-Building

The piece I'm most excited about technically is the **Scenario Generator**. Mid-demo, the host or a stream-audience member can pitch a breach in one or two sentences — *"An employee's session token gets stolen from a browser cache at a conference and used to exfiltrate regulatory submission docs"* — and ~40 seconds later a brand new scenario is hot-loaded into the game.

The agent emits structured JSON (premise narration, 5 suspects with roles + hidden truths, 6-10 pieces of evidence, 4-6 violated controls, a clue graph, and a compliance lesson), which then goes through a validation layer:

```python
try:
    merged_scenario = load_scenario_from_dict(scenario_override)
except ScenarioValidationError as exc:
    if validation_attempt < max_validation_retries:
        # Feed the validation error back to the model as a corrective message
        user_message = _build_validation_retry_message(
            breach_description, exc
        )
        continue
    raise
```

If validation fails (wrong perpetrator count, missing canonical suspect, malformed evidence reference), the generator retries with the specific error fed back as a corrective message. Two validation cycles is usually enough. The session state then resets cleanly, the new briefing renders, and the player can start investigating immediately. **Same agents, brand new world.**

---

## 💡 Lessons from Building It

A few things I'd carry to the next reasoning-agent project:

| Lesson | What I'd Do Differently |
| --- | --- |
| **Ground anything compliance-related in retrieval, always** | Even "general knowledge" policy citations are a hallucination risk |
| **The activity log was the secret weapon** | Plan the observability surface as a first-class deliverable, not an afterthought |
| **Validation + corrective retry is more robust than perfect prompting** | Let the model fix its own structured output errors when possible |
| **Connected Agents pattern works for clear role separation** | Don't try to make one agent do everything; specialize and route |
| **Keep one terminal-driven backup surface** | When the stream demo gods are angry, falling back to a CLI is a graceful recovery |

---

## 🎯 Why Microsoft Foundry Agent Service Was the Right Fit

I wanted Foundry specifically because:

- **Connected Agents** maps cleanly onto a party-of-agents game design
- **Model Router** gives me one endpoint that picks the right backing model per agent
- **Azure AI Search** with agentic retrieval is first-class
- **Azure Entra ID** integration means the whole thing runs under enterprise SSO with no token juggling
- **Streaming responses** from Azure OpenAI deployments work cleanly with Python's async generators

The full stack — Foundry Agent Service + Azure OpenAI + Azure AI Search + Azure Storage — sits inside one resource group (`SCH_AI_POC`), which made tearing it down between dry runs trivial.

---

## 🧪 Try It Yourself

Clone the repo and follow the README:

```bash
git clone https://github.com/lwhieldon/msft-enterprise-learning-agent.git
cd msft-enterprise-learning-agent
python -m venv .venv
.\.venv\Scripts\activate     # Windows
pip install -r requirements.txt

# Set up .env with your Azure credentials, then:
chainlit run app.py -w        # the player-facing UI
.\scripts\tail_activity.ps1   # the orchestration evidence
```

The default scenario (*Breach at Helix Dynamics*) ships with all three pre-built cases (Default, Supply Chain, Vishing), and the Generate button creates new ones live.

---

## 💬 What's Next

A few directions I'm thinking about:

- **Real token streaming in the Chainlit UI** (currently the audience waits for full responses then sees them appear; live token streaming would feel even more alive)
- **Voice mode** with Azure Speech, so the player can actually interview suspects
- **Multi-player** with each player taking a role on the investigation team
- **Custom compliance domains** — same engine, different policy library (financial services, healthcare, manufacturing)

If you build something on top of this, want to compare notes on multi-agent design, or just want to swap reasoning-agent war stories, drop me a message on [LinkedIn](https://www.linkedin.com/in/lee-w-9b3a0620/) or [GitHub](https://github.com/Lwhieldon).

Special thanks to [Lee Stott](https://www.linkedin.com/in/leestott/) and [Carlotta Castelluccio](https://www.linkedin.com/in/carlotta-castelluccio/) at Microsoft for hosting the Reactor battle and giving these projects a stage. 🎤

---

*This site is open source. [Improve this page](https://github.com/Lwhieldon/Lwhieldon.github.io/edit/main/_posts/2026-06-10-compliance-academy.md).*
