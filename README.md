# ShopAssist

**A reliability-first, domain-aware AI agent for e-commerce customer support.**

ShopAssist is designed to resolve customer questions with grounded evidence, explicit policy controls, and safe human escalation. It prioritizes correct, verifiable outcomes over answering every request. When the available evidence is insufficient, the request is ambiguous, or the action is outside the agent's authority, ShopAssist chooses a controlled next step instead of guessing.

> **Project positioning:** ShopAssist combines domain knowledge, evidence retrieval, program-controlled decisions, and safe escalation to deliver a reliable e-commerce AI agent.

## Project Overview

E-commerce support questions often involve store-specific policies, order context, sensitive information, and legal or payment requirements. A general-purpose language model may produce fluent but unsupported answers in these situations.

ShopAssist addresses this challenge with a workflow that retrieves relevant knowledge, evaluates evidence, applies explicit decision rules, and validates the generated response. The design aims to make answers traceable and to route cases to clarification or human support when automated handling is not appropriate.

## Core Design Principles

- **Reliability over reach:** Optimize for correctly resolving a request, not maximizing the number of responses. Abstain when reliable evidence is unavailable.
- **Evidence-grounded generation:** Base customer-facing claims on retrieved, attributable sources.
- **Program-controlled decisions:** Use explicit workflow rules to determine the handling state, especially for sensitive or high-impact requests.
- **Domain-aware reasoning:** Incorporate e-commerce policies and relevant Taiwan regulatory and security knowledge into retrieval and decision-making.
- **Safe escalation:** Ask for necessary information or hand off to a human when a request exceeds the agent's evidence or authority.
- **Auditability and traceability:** Preserve decision states, evidence references, validation outcomes, and handoff reasons for review.

## Four Core Highlights

### 1. Reliability First

ShopAssist does not answer simply to avoid leaving a question unanswered. It selects `ABSTAIN` when evidence is inadequate, `CLARIFY` when key information is missing, and `HANDOFF` when human authority or judgment is required.

### 2. Evidence-Grounded AI

The retrieval design combines **BM25 lexical search** with **dense semantic retrieval**, then merges rankings using **Reciprocal Rank Fusion (RRF)**. Evidence gates assess whether the retrieved material is sufficient before generation. After generation, citation and numeric checks help ensure that claims and important figures are supported by the sources.

### 3. Domain-Aware Reasoning

The knowledge and decision workflow is designed to account for e-commerce requirements, including Taiwan's **Consumer Protection Act**, store return and exchange policies, **PCI DSS**, and personal data protection. These topics inform the agent's handling and escalation decisions.

### 4. Controlled and Auditable Decisions

Application logic governs the outcome state. Responses can be traced through their retrieved sources, citations, validation results, and handling labels. Handoffs can include reason codes to support consistent operations and later review.

## High-Level Architecture and Workflow

```text
Customer request
      |
      v
Understand intent and identify missing context
      |
      v
Retrieve domain and store knowledge
(BM25 + Dense retrieval -> RRF fusion)
      |
      v
Evaluate evidence gates and policy constraints
      |
      +--> Insufficient evidence ------------> ABSTAIN
      +--> Missing required information -----> CLARIFY
      +--> Outside agent authority/risk ------> HANDOFF
      |
      v
Generate an evidence-grounded response
      |
      v
Verify citations and numeric claims
      |
      +--> Checks pass -----------------------> ANSWERED
      +--> Checks fail / evidence inadequate -> ABSTAIN or HANDOFF
```

This diagram describes the conceptual design. Exact routing thresholds and implementation details depend on the system configuration.

## Decision States

| State | Meaning |
| --- | --- |
| `ANSWERED` | The request can be answered with sufficient supporting evidence and the response passes validation. |
| `ABSTAIN` | The agent cannot provide a reliable answer from the available evidence. |
| `CLARIFY` | The request needs additional customer information before it can be handled safely. |
| `HANDOFF` | The request requires a human agent or authority beyond the agent's scope. |

These states are selected by the controlled workflow rather than being left solely to free-form model output.

## Key Technologies and Concepts

- Hybrid retrieval: **BM25 + dense retrieval + RRF**
- Pre-generation **evidence gates**
- Post-generation **citation and number verification**
- Domain knowledge for the Taiwan **Consumer Protection Act**
- Store-specific **return and exchange policies**
- **PCI DSS** and **personal data protection**
- Explicit decision states, safe escalation, and auditable handling

## Documentation

- [AI Agent design principles and four core highlights](docs/conversation-ai-agent-design.md)
- [ShopAssist proposal document](docs/ShopAssist_Proposal_Document.md)
- [ShopAssist project analysis](docs/ShopAssist_project_analysis.md)
- [System design](docs/DESIGN.md)

## Project Scope

This README summarizes the ShopAssist design described in the project materials. It does not claim that every component described here is already implemented or deployed.
