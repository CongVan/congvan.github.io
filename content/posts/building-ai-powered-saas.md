---
title: "Lessons from Building an AI-Powered SaaS from Scratch"
date: 2026-04-02T01:00:00+07:00
draft: false
tags: ["Backend", "AI Research", "DevOps"]
categories: ["Engineering"]
summary: "What I learned taking an AI conversational platform from first commit to 250K+ monthly interactions serving enterprise clients."
ShowToc: true
---

Over the past three years, I built an AI-powered SaaS platform from the ground up as the founding engineer. The platform automates structured business conversations -- VC screening, customer support, booking -- for 20+ enterprise clients, including a venture capital firm.

Here are the key lessons from that journey.

## Start with the State Machine, Not the AI

The biggest mistake teams make with AI products is letting the LLM drive the conversation flow. We built what we called the **Agenda Engine** -- a proprietary state machine that forces AI through defined workflows:

1. **Collect Data** -- Gather structured information from the user
2. **Verify** -- Confirm the data is complete and valid
3. **Action** -- Execute the business logic

This gave us **deterministic reliability** in high-stakes enterprise interactions. The AI handles natural language; the state machine handles the business logic. They never cross responsibilities.

## Decouple Early

One of the best architectural decisions we made was decoupling the core application from the AI processing layer. This let our AI team iterate on model tuning independently while the platform team scaled infrastructure.

The boundary was simple: the platform sends structured prompts and receives structured responses. Everything in between is the AI team's domain.

## Monorepo for Speed

We used a monorepo with:
- **Next.js** (App Router) for dashboards
- **Node.js/Express** for the API
- **PostgreSQL + Redis + BullMQ** for data and async processing

Shared types across frontend and backend eliminated an entire class of bugs. One PR could update the API contract and the consuming UI simultaneously.

## Ship Weekly

We maintained a weekly ship cadence with a serverless-first strategy on GCP Cloud Run. Auto-scaling and zero-downtime releases meant we could push changes confidently.

The key enabler: a solid CI/CD pipeline with GitHub Actions that ran type checks, tests, and automated deployments on every merge to main.

---

More deep dives on specific parts of this architecture coming soon.
