# Building a Domain-Specific AI Workflow for HTS Classification and Duty Determination

Import and customs teams face a recurring problem: turning a plain-language product description into the correct **Harmonized Tariff Schedule (HTS) code** and the right **duty rate** — fast, consistently, and with an explanation auditors can follow.

Generic LLMs are not enough. Tariff lookup is a chain of specialized decisions spread across thousands of codes, multiple duty columns, trade programs, and country-specific overlays. This project builds a **guided workflow** backed by **two fine-tuned domain models**, each trained for one job, orchestrated through a simple web app.

---

## The business case

### What importers and trade analysts need

Every shipment raises the same questions:

1. **What is the correct HTS code?** — Wrong classification drives wrong duty, compliance risk, and delays.
2. **What duty applies?** — General rate, special program, Column 2, Section 301, or a combination?
3. **Why?** — Operations, legal, and customs brokers need a clear basis, not just a number.

Manual lookup in PDF tables and spreadsheets is slow and error-prone. A single misread column, for example applying Column 2 to a European origin, can change the rate dramatically.

### What this app delivers

| Business need | How the app addresses it |
|---------------|---------------------------|
| **Faster triage** | Vector search returns the top 4 HTS candidates in seconds from a product description |
| **Expert classification** | A fine-tuned **Qwen** model reasons over candidates and picks the best leaf code |
| **Accurate duty** | A fine-tuned **Llama duty LoRA** produces structured duty output; totals are validated against HTS table logic |
| **Explainability** | Every step shows reasoning, breakdown, and raw model output for review |
| **Repeatable process** | Same three-step flow for every product — retrieve → classify → duty |

The goal is not to replace a licensed customs broker, but to **compress research time** and **standardize** how HTS and duty questions are answered inside the organization.

---

## Solution overview

The app is designed around a simple workflow that mirrors how trade teams evaluate HTS and duty questions:

**retrieve → classify → compute duty**

A user starts with a plain-language product description. The system retrieves the most relevant HTS candidates, uses a fine-tuned Qwen model to reason over the candidate list, and then uses a fine-tuned Llama duty model together with deterministic HTS logic to produce a duty breakdown and explanation.

This split is intentional. A single model would need to master both **open-ended product matching** and **rule-bound rate arithmetic** — making it harder to train, harder to trust, and more expensive to serve. Separating classification from duty determination matches how trade teams actually work and allows each model to be improved independently.

<INSERT DEMO VIDEO HERE>

The demo shows the full flow:

1. Entering a product description  
2. Retrieving candidate HTS codes  
3. Running classification reasoning  
4. Confirming the HTS code and origin  
5. Computing duty with a validated total, breakdown, and explanation  

Under the hood, the workflow combines:

- **Retrieval** over the official HTS catalog using embeddings and a vector store  
- **Classification** with a Qwen LoRA trained on product → candidate → HTS reasoning examples  
- **Duty determination** with a Llama LoRA trained on tariff-field → duty outcome scenarios from 2026 HTS Rev 8 data  
- **Deterministic validation** so HTS table fields remain authoritative for the final duty number  

Heavy inference runs in the cloud: Tinker for Qwen, RunPod/vLLM for duty. The UI, API, and vector search run locally or on a lightweight server.

---

## Fine-tuning model 1: HTS classification — Qwen

### Domain task

**Input:** A product description plus a short list of retrieved HTS candidates: code, description, and rate hints.  
**Output:** Reasoning and a final **leaf-level HTS code** chosen from the candidate list.

This mirrors how a trade analyst works: narrow the search, then apply product knowledge to pick the right line.

### How training data is built

`generate_hts_lora_reasoning_dataset.py` creates examples from the normalized HTS leaf table:

1. Take a real HTS row and synthesize product description variants.  
2. Run **vector-store retrieval** to build a realistic **Candidates** block, including ambiguous cases.  
3. Label the assistant response with the **ground-truth HTS code** and reasoning.  

The model learns to:

- Compare product wording against candidate descriptions  
- Respect the candidate list at inference time  
- Explain when information is insufficient for a final classification  

### Why Qwen for this step

Classification requires **long-context reasoning** over multiple similar tariff lines. Qwen 3 4B Instruct is a strong fit for structured comparison and explanation — a different skill than computing a duty percentage from known fields.

---

## Fine-tuning model 2: Duty determination — Llama

### Domain task

**Input:** A **fixed** HTS row — code, description, general/special/Column 2 rates, additional duties, origin, trade program, and eligibility.  
**Output:** A structured duty answer:

```text
Total duty: 25.0%

Breakdown:
- Base duty (general rate): Free
- Special rate applied: none
- Column 2 rate applied: none
- Section 301 applies (additional duty: 25%)

Reasoning: ...
```

Duty is a **rules-heavy** problem: the model must apply the right column and overlays, then explain clearly.

### How training data is built — v4 pipeline

`build_duty_dataset_2026.py` orchestrates:

1. **Normalize** official `htsdata.xlsx`, 2026 HTS Rev 8, into leaf rows.  
2. **Generate scenarios** per HTS line — general rate, Column 2 origin, special program, Section 301 for China, and missing information.  
3. **Compute labels deterministically** with `duty_training_logic.py` so Total duty and Breakdown are always correct.  
4. **Attach templated reasoning** in consistent prose for the assistant message.  

Scenario types in the dataset:

| Scenario | Business meaning |
|----------|------------------|
| General rate | Normal import from a Column 1 country |
| Column 2 | Embargoed / restricted origins |
| Special program | Preferential rate when program + eligibility confirmed |
| Section 301 | China/HK additional duties on top of base rate |
| Missing info | Cannot finalize duty without origin or program details |

### Why Llama 3.2 1B for this step

Duty inference runs at high volume on **RunPod serverless with vLLM**. A 1B base keeps cost and latency low; the LoRA carries the domain-specific behavior. The prompt format is a single flat user message, not chat templates — exactly how the model was trained.

### Trust layer in production

Duty **percentages** in the UI come from the same deterministic logic used to build training labels. The LoRA drives narrative; the **HTS table fields in the prompt** are authoritative for the final number. That hybrid design prevents hallucinated rates, for example 15% vs 25% on Section 301, from reaching users.

---

## Summary

This project demonstrates a **practical pattern for regulated domains**:

1. **Retrieve** from an authoritative catalog using embeddings and vector store.  
2. **Classify** with a fine-tuned reasoning model, Qwen LoRA.  
3. **Compute** with a second fine-tuned model, Llama duty LoRA, plus deterministic validation.  
4. **Explain** every step in language compliance teams can review.  

The business value is speed and consistency: less time in tariff tables, fewer wrong-column mistakes, and a repeatable audit trail from product description to duty outcome — powered by two purpose-built domain models, not one general chatbot.

---

*Technical setup notes, including local development, RunPod, and API keys, are in [BLOG.md](./BLOG.md).*
