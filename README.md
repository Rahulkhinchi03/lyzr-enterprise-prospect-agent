# Building an Enterprise Prospect Research Agent on Lyzr Studio

Check out on the App store: https://studio.lyzr.ai/create-new-agent/6a84a4e6d468c8c0411988c7?tab=playground&public=true

**A managerial-orchestration cookbook, including what broke and how I found it.**

---

## TL;DR

I built a four-agent system on Lyzr Studio that takes a target company name and returns a full enterprise prospect research brief: firmographics, buying signals, likely stakeholders, pain points mapped to specific Lyzr capabilities, an outreach angle, and discovery questions. One Manager delegates to three specialist workers, one of which is grounded in a Knowledge Base built from Lyzr's own product content.

Getting all four agents built and wired took a fraction of the effort. Almost everything else here went into testing it against real companies, and that's the part of this cookbook worth actually reading. I caught a hallucinated headcount, a knowledge base with duplicate and missing sources, a citation format that looked trustworthy but wasn't, and a compilation step that silently dropped the one safety mechanism I'd built. I also flagged something as a fabrication that turned out to be true, twice, because my own verification process was stale. That mistake is in here too, because it's a better lesson than any of the ones where I was right.

---

## Why Managerial Orchestration Fits This Problem

The assignment specified managerial orchestration, so this wasn't an open choice, but it's worth being clear about why it's the right one for this problem anyway, not just the given one. Lyzr Studio also offers SuperFlow, a fixed, directed workflow, good when every run follows the same sequence of steps. Managerial orchestration is dynamic: a Manager Agent interprets a goal, decomposes it, and delegates to worker agents at runtime.

Prospect research doesn't have a fixed shape. Researching a 200-year-old public bank and a two-year-old fintech pulls from different sources, surfaces different signals, and needs different judgment calls about what's actually relevant. That's what managerial orchestration handles and a fixed SuperFlow graph doesn't. SuperFlow would still be the right call for something like a fixed KYC intake pipeline, where every applicant goes through the same steps in the same order, that's just not this problem.

---

## Architecture at a Glance

```
User: "Research [Company] as a prospect for Lyzr"
        |
        v
Enterprise Prospect Research Manager
        |
        +--> Company Intelligence Agent
        |       (firmographics, recent news, composio_search)
        |
        +--> Signal & Stakeholder Mapping Agent
        |       (buying signals, likely evaluators, composio_search)
        |
        +--> Positioning & Brief Writer Agent
                (maps pain points to Lyzr capabilities,
                 grounded in an 8-source Knowledge Base)
        |
        v
One consolidated brief:
Company Snapshot, Buying Signals, Likely Stakeholders,
Mapped Pain Points & Lyzr Fit, Outreach Angle, Discovery Questions
```

The Manager doesn't do any research itself. It delegates, waits for both workers to return, passes their combined output to the Positioning agent, then compiles everything into one document. Company Intelligence and Signal & Stakeholder Mapping run independently, both grounded in live web search. Positioning is the only agent with Knowledge Base access, and it only synthesizes, it never searches the web itself.

<img width="1512" height="826" alt="Dashboard" src="https://github.com/user-attachments/assets/f7a00ab0-902e-4f9c-9a35-0679df2959b8" />
---

## Prerequisites

- A Lyzr Studio account (studio.lyzr.ai)
- A web-search tool connector available in your Tools catalog. I used `composio_search`.
- Awareness of your credit budget. Every full pipeline run calls four agents plus a knowledge base retrieval in one shot, that adds up fast on a metered plan. Test agents individually where you can, save full pipeline runs for validation, not iteration.

---

## Part 1: The Manager

Any single agent in Lyzr Studio becomes a Managerial Agent the moment you attach children to it via the **+Agent** button. There's no separate "create a manager" flow, you build a normal agent, then give it a team.

**Role**
> You are the Enterprise Prospect Research Manager for Lyzr's go-to-market team. You receive a target company name and optional deal context, and orchestrate a team of specialist research agents to build one complete, accurate prospect brief.

**Goal**
> Produce one consolidated Enterprise Prospect Research Brief per request, ready for a Solutions Engineer or DevRel rep to use in outreach. Never present a claim that none of your worker agents actually found.

**Instructions**
> 1. Take the target company name and any deal context given.
> 2. Delegate to Company Intelligence Agent for firmographics and recent news.
> 3. Delegate to Signal & Stakeholder Mapping Agent for tech and hiring signals plus likely buyer personas.
> 4. Pass both outputs to Positioning & Brief Writer Agent.
> 5. Compile the final brief with these sections: Company Snapshot, Buying Signals, Likely Stakeholders, Mapped Pain Points & Lyzr Fit, Suggested Outreach Angle, Discovery Questions. In the Mapped Pain Points & Lyzr Fit section, keep the Positioning & Brief Writer Agent's (Source: X) citations exactly as given, word for word, do not remove them, summarize them away, or paraphrase them out during compilation.
> 6. If any section can't be verified, write 'Not found, needs manual verification' instead of guessing.
> 7. Never invent numbers, names, or quotes.

That step 5 clause about preserving citations wasn't in the first version. It's there because of a bug I'll walk through later, one that only shows up once you test what the Manager does with a worker's output, not just what the worker itself produces.

- **Model:** gpt-5.4-mini. 
- **Features:** Memory on (Lyzr Cognis, on by default, this lets you refine a brief with a follow-up in the same session). Responsible AI was left off, not by choice, it's plan-gated behind a paid tier on Community accounts. More on how I compensated for that below.
- Leave Knowledge and Tools empty on the Manager. Those live on the workers.

<img width="1512" height="825" alt="Manager Agent" src="https://github.com/user-attachments/assets/6b1ff45e-d695-4828-a05f-adf503346eee" />
---

## Part 2: Company Intelligence Agent

Built as a native child via +Agent under the Manager.

**Role**
> You are an enterprise company research analyst. Given a company name or domain, return only objective, sourced firmographic and news information.

**Goal**
> Return a firmographic snapshot: industry, HQ, employee size band, funding or ownership stage, and any public news from the last 12 months about technology, AI, or digital transformation initiatives.

**Instructions**
> Use your connected web-search tool for every fact. Name the source for each claim. If something can't be confirmed, write 'Not publicly available.' Never fabricate financial figures, headcounts, or executive names. Only state a specific number (headcount, revenue, funding, a date tied to a figure) if it appears directly in your tool's retrieved results for this exact query, never from memory or general knowledge. If the retrieved results don't contain a number for something, say 'not found in retrieved results' instead of stating one.

That second half of the instructions, the part about only stating a number that appears verbatim in the retrieved results, is the result of catching this agent hallucinate a headcount twice before it got it right. That's in the testing section.

- **Tools:** `composio_search`. 
- **Model:** gpt-5.4-mini, a fast, cheap model is fine here because the accuracy comes from the search tool doing the grounding, not from the model's own reasoning. Save your strongest model for the agents doing actual synthesis.
- **Managerial Context** (the field that tells the Manager when to call this agent, separate from the agent's own instructions):
> Call this first for any target company. Give it the company name or domain. It returns sourced firmographics and recent tech/AI news, nothing else. Always call it before Positioning & Brief Writer Agent.

<img width="1512" height="827" alt="Agent1" src="https://github.com/user-attachments/assets/73c6ddb3-0ab4-4769-97ce-c95abe889c41" />
---

## Part 3: Signal & Stakeholder Mapping Agent

**Role**
> You are a B2B buying-signal and org-mapping specialist for enterprise AI and software sales.

**Goal**
> Identify public signals that this company is investing in AI or automation (careers page postings, engineering blog mentions, GitHub activity, conference talks), and the job functions most likely to own evaluating a platform like Lyzr, e.g. Head of AI, VP Engineering, Head of Support Ops, Chief Data Officer, based on typical org structures for a company of this size and industry.

**Instructions**
> Use your connected web-search tool for every claim. Name the source for each one. If something can't be confirmed, write 'Not publicly available.' Never invent named individuals, titles, or contact details, only report a named person if their name and title appear directly in your tool's retrieved results. Only state a specific fact (a hiring signal, a title, a team size, a date) if it appears directly in the retrieved results for this exact query, never from memory or general knowledge.

- **Tools:** same `composio_search` connector. 
- **Managerial Context:**
> Call this for the same target company, either right after or in parallel with Company Intelligence Agent. It returns public buying signals and the likely stakeholders or named evaluators at that company. Always call it before Positioning & Brief Writer Agent.

Tested against HDFC Bank and Axis Bank, two companies with very different press coverage density. On Axis Bank it correctly named a real, verifiable Chief AI Officer and CISO, both confirmed independently outside the platform. On HDFC Bank in a separate run, it correctly stayed generic and said stakeholder names weren't confirmed, because that run's search results didn't surface the same quality of named data. Same instructions, two different but both honest outcomes. That's the behavior you want: precision when the data supports it, restraint when it doesn't.

<img width="1512" height="826" alt="Agent2" src="https://github.com/user-attachments/assets/04a86835-85c6-437a-b345-2b77d56d981d" />
---

## Part 4: The Knowledge Base

This is the part that determines whether the Positioning agent is actually useful or just makes things up with better formatting.

**Type:** Basic (simple vector retrieval). Graph and Semantic Data Model weren't relevant, there's no entity-relationship structure or live database here, just a curated set of documents.

**Name:** `lyzr_product_positioning_kb`

**Sources (8, added individually via Add Source > Add Website, not Add Live Source):**
- `https://docs.lyzr.ai/enterprise/get-started/intro`
- `https://docs.lyzr.ai/introduction/key-concepts/manager-agent`
- `https://docs.lyzr.ai/introduction/using-lyzr/components`
- `https://docs.lyzr.ai/introduction/getting-started/intro`
- `https://www.lyzr.ai/case-studies/japanese-bank-automated-customer-onboarding/`
- `https://www.lyzr.ai/playbook/banking/`
- `https://www.lyzr.ai/banking-agents/ai-customer-service-agent/`
- `https://www.lyzr.ai/blog/ai-in-customer-service-for-banks`

I split these into two groups on purpose. The first four cover what Lyzr Studio and the Manager Agent pattern actually are, general platform grounding. The last four are banking-specific, because both of my test accounts are banks, and that's where the real leverage is: a real customer case study (OCR, verification, managerial coordination, Temenos integration, SOC2/ISO27001/HIPAA), a full banking playbook with named pre-built agents (Regulatory Compliance Audit Agent, Cash Flow Prediction Agent, Invoice Payment Agent, Payment Reconciliation Agent, and more), and product pages describing Lyzr's actual banking-specific architecture.

**Why Add Website and not Add Live Source:** Live Source is a recurring crawler, by default it'll crawl three link-levels deep across an entire doc section and re-sync daily. Pointed at `docs.lyzr.ai/enterprise/`, that means indexing far more than the four pages I actually wanted, and re-embedding it every 24 hours whether it changed or not. Add Website with a single URL gets exactly the page I chose, once. Worth knowing before you start, since the two options look similar but behave very differently at scale.

**Retrieval type on the consuming agent:** Basic. There's only one KB, so One Shot's multi-KB planning step and Agentic's ReAct tool loop both add cost with nothing to plan across.

<img width="1512" height="827" alt="kb" src="https://github.com/user-attachments/assets/7c60b28e-a913-4d07-ba38-5d005fe55cf8" />
---

## Part 5: Positioning & Brief Writer Agent

This is the agent the whole build exists to support, and it's the one that broke the most times during testing. Its final instructions are the product of three separate rounds of fixes.

**Role**
> You are a Lyzr solutions engineer who turns raw prospect research into a positioning-ready brief, grounded strictly in Lyzr's real, documented product capabilities.

**Goal**
> Using the Company Intelligence and Signal/Stakeholder outputs plus the attached Knowledge Base, identify 2 to 3 likely pain points this company has that Lyzr's platform addresses, each tied to a specific capability found in the Knowledge Base. Draft one paragraph suggesting an outreach angle, and 2 to 3 discovery-call questions.

**Instructions**
> Only reference Lyzr capabilities, products, or case studies that are directly retrieved from the attached Knowledge Base for this query, never from general knowledge or assumption, even if it sounds plausible or matches a real product name you know from elsewhere, competitor products in particular. End every capability you cite with a source tag using ONLY one of these exact names, nothing else is valid: (Source: Banking Playbook), (Source: Japanese Bank Case Study), (Source: Customer Service Agent Page), (Source: Customer Service Blog), (Source: Enterprise Intro), (Source: Manager Agent Docs), (Source: Studio Components Docs), (Source: Getting Started Docs). If you cannot match a capability to the retrieved content from one of these exact eight sources, do not name it, do not describe it, and do not cite it, say the pain point has no confirmed Lyzr capability match instead. When the Knowledge Base does return relevant content, be as specific as possible: name the exact agent, module, or product term it uses (for example 'Regulatory Compliance Audit Agent,' 'AgentMesh,' 'Cross-border Payment Optimization Agent'), don't flatten it into a generic paraphrase like 'workflow orchestration.'

- **Knowledge:** `lyzr_product_positioning_kb`, retrieval type Basic. 
- **Model:** same tier as the Manager, this agent maps pain points to capabilities, that's judgment, not extraction, and it deserves the stronger model. 
- **Managerial Context:**
> Call this last. Pass it the full outputs from both Company Intelligence Agent and Signal & Stakeholder Mapping Agent, it needs both to map pain points and draft the outreach angle. Never call it first or with only one of the two prior outputs.

<img width="1512" height="827" alt="Agent3" src="https://github.com/user-attachments/assets/9f46d49a-1313-4c30-80b0-59196457d07f" />
---

## Part 6: Wiring the Manager

Once all three children exist, attach them to the Manager via +Agent, not +A2A. A2A is Google's Agent2Agent protocol, for importing an already-deployed external agent from outside your workspace. All three of these are native children built in the same Studio project, so +Agent is correct.

<img width="1511" height="825" alt="Managerial" src="https://github.com/user-attachments/assets/b7b12ddb-4ea3-4b83-89cb-655b1469ded5" />

Set the Managerial Context field on each child (shown above), this is what tells the Manager when to delegate to each one and in what order. It's a separate field from the child's own Role and Goal, easy to miss, and without it the Manager only has its own numbered instruction list to go on for sequencing, which is thinner protection than having both.

---

## Deploying and Sharing It

Every agent in Lyzr Studio deploys automatically behind a REST API endpoint the moment it's created, no separate hosting step needed. The Deploy tab gives you the exact cURL command to call it directly.

Don't hand that cURL to anyone outside your own use. The `x-api-key` in it is tied to your account and your credits. Anyone who has it can run inference against your account, not just this one agent.

For giving someone else access, Deploy has two separate paths, and they do different things.

- **Share** invites specific people by email to one specific agent, without publishing it anywhere or adding them to your organization. This is the right tool for letting one person try it.

- **Publish** launches the agent as an app with a visibility choice: Team, visible only inside your own Lyzr organization, or App Store users, public and listed on Lyzr's marketplace for anyone to find and run. Publishing also asks for an app name, description, welcome message, category, industry, and function, all shown to whoever opens it.

One thing worth knowing before you publish: if you have a Manager with worker agents underneath it, publish the Manager, not one of the workers. Publishing a single worker in isolation only exposes that one piece, not the orchestration that's the actual point of the build.

<img width="1512" height="827" alt="Deploy" src="https://github.com/user-attachments/assets/6a545acd-3c07-4a6f-a315-3ea114534d93" />

App names need to be unique across the platform. A generic name like "Enterprise Prospect Research Manager" may already be taken by someone else's agent. Appending your own name resolves it without a rewrite.

---

## Known Limitations, Documented on Purpose

- **Responsible AI is plan-gated** on the Community tier used for this build. Compensated by moving the anti-fabrication rules (source citation, verbatim-only numbers, "not found" fallbacks) directly into each agent's Instructions text instead of relying on a platform-level groundedness check.
- **Model choice is uniform** (gpt-5.4-mini across all four agents) rather than upgraded for the two synthesis-heavy agents. Every test passed on this tier, so the credit-efficient choice is the empirically justified one, not a compromise.

---

## Sample Output

Abbreviated excerpt from a validated run, `Research HDFC Bank as a prospect for Lyzr`, after all fixes above:

> **Mapped Pain Points & Lyzr Fit**
>
> Need to scale GenAI beyond experimentation into governed deployment
>
> Lyzr fit: Agent Studio is a low-code/no-code platform to build and deploy autonomous AI agents rapidly, and it is positioned as safe-by-design with built-in audit logs, role-based access, hallucination control, and compliance-first design. (Source: Banking Playbook)
>
> Need for workflow automation across back-office and operational processes
>
> Lyzr fit: The Banking Playbook emphasizes end-to-end workflow ownership and gives examples such as Invoice Payment Agent and Payment Reconciliation Agent, both framed as multi-agent workflows that automate approvals, exception handling, and reconciliation. (Source: Banking Playbook)

Both claims checked against the actual playbook content: verbatim accurate.

---

## Where This Goes Next

- **Upgrade retrieval type to Agentic** if Bug 4's thin-match rate becomes a real problem at higher usage, trading credit cost for retrieval robustness.
- **Add an ICP knowledge base** alongside the product KB, so the Positioning agent can weigh fit against Lyzr's actual ideal-customer profile, not just capability match.
- **Extend past banking.** The architecture is industry-agnostic, only the KB sources are banking-specific right now. Swapping in insurance or healthcare sources would need no changes to any agent's Role, Goal, or core Instructions.
- **Schedule it** against a real target-account list using the Manager's Automation tab, not built here, but the delegation logic doesn't change.

---

## Appendix: Full Config Reference

All exact Role, Goal, Instructions, and Managerial Context text for all four agents is reproduced verbatim in the sections above, ready to copy-paste into Lyzr Studio's Build tab fields. Knowledge Base source list is in Part 4.
