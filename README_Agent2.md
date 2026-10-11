# 🛒 Olist Ontology Agent

An AI agent that answers questions about a real e-commerce dataset (the Olist Brazilian marketplace) by querying a structured **ontology** of the business instead of guessing. It can change data only through rule-checked, human-approved actions, and every step is written to an audit log.

This is the second project in a learning series on ontologies and AI agents. The first was a small retail agent on hand-made data. This one uses a real, multi-table dataset (about 100,000 orders), so the links between tables matter.

Built with Python, SQLite and the Gemini API. The ontology idea is inspired by Palantir Foundry/AIP, used here for reference only.

---

## What the code does

```
CSV files ──▶ SQLite database ──▶ Ontology ──▶ Tools ──▶ Gemini agent ──▶ you
 (Kaggle)      (cleaned)        objects,       7 narrow    asks, never
                                links,         functions   touches data
                                actions
                                   └──── every change needs your approval ────┘
```

1. **Load and clean** the dataset CSV files into a local SQLite database (dates normalized, ids made unique, category names translated to English).
2. **Model the business** as objects, links and actions (the ontology).
3. **Expose seven narrow tools** to the AI. It never gets direct access to the data.
4. **Let the agent ask questions** through those tools. Changes pass rule checks and a human "yes" first.
5. **Log everything** (questions, tool calls, approvals, changes) to `audit_log.jsonl`.

---

## Why an ontology helps the agent

| Benefit | How it shows up here |
|---|---|
| **Grounded answers** | Numbers come from tool calls on real data, not from the model's memory |
| **Multi-step reasoning** | Links let it go from late orders to the seller to the reviews |
| **Safe actions** | Rules inside each action refuse bad requests, such as a refund larger than the amount paid |
| **Manageable scale** | Summary tools and a 25-row cap keep the model from drowning in 100,000 orders |
| **Traceability** | The audit log shows what was asked, what ran, and who approved what |

---

## The ontology

**Object types (7):** Customer, Order, OrderItem, Product, Seller, Review, Payment

**Links (12), examples**

| From | Link | To |
|---|---|---|
| Order | `customer` | Customer |
| Order | `items` | OrderItem (many) |
| Order | `payments` / `reviews` | Payment / Review (many) |
| OrderItem | `product` / `seller` / `order` | Product / Seller / Order |
| Seller | `items` | OrderItem (many) |
| Product | `items` | OrderItem (many) |
| Customer | `orders` | Order (many) |

**Computed properties.** `is_late` and `delivery_days` on Order are calculated by a database view, not stored, so they cannot go stale. An order is late only if it was delivered after the estimated date.

**Actions** (all need human approval; they write only to separate tables and never edit the original dataset)

| Action | Rules checked |
|---|---|
| `flag_order_for_review` | Order exists; reason is 3 to 200 characters; the order is not already flagged |
| `propose_refund` | Amount is positive; cannot exceed what was paid minus earlier proposals. It is a proposal only, no money moves. |
| `add_seller_note` | Seller exists; note is 3 to 300 characters |

Each action is first **validated as a dry run**, then approved, then applied inside one database transaction (all or nothing).

---

## The tools the AI can call

| Tool | Purpose |
|---|---|
| `search_objects` | Find objects by field, with `= != < <= > >= contains`; `null` finds missing values |
| `get_linked_objects` | Follow a link, for example Order → items |
| `delivery_stats` | Late-delivery rates, overall or by state or month (states need 50+ delivered orders) |
| `seller_performance` | Rank sellers by late rate, average review or order count (default minimum 30 orders) |
| `sales_summary` | Revenue in BRL by category, month or state (canceled orders excluded) |
| `data_quality_report` | Counts of known data problems |
| `execute_action` | The only way to change anything, with approval |

Safety design: fields, operators and tables are checked against whitelists, and values are passed as parameters, so the model cannot write its own SQL.

---

## Getting started

### Requirements
- Python 3.10+
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) (keys may start with `AQ.` or `AIza`)
- The Olist dataset from Kaggle (see Data below)

### 1. Try it without the real data
```bash
pip install -U google-genai pandas
python olist_agent.py --selftest                 # 43 checks, no AI and no key needed
python olist_agent.py --make-sample sample       # fake data in Olist's file format
python olist_agent.py --data sample --db sample.db
```

### 2. Use the real dataset
Download it from Kaggle: <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce>

In Colab you can also use `kagglehub` (if it asks for Kaggle credentials, follow its prompt):
```python
!pip install -U google-genai pandas kagglehub
import kagglehub, shutil
path = kagglehub.dataset_download("olistbr/brazilian-ecommerce")
shutil.copytree(path, "olist", dirs_exist_ok=True)
```
Then run the agent (the first run builds `olist_agent.db`):
```python
%run olist_agent.py --data olist
```
Later runs reuse the database. Use `--rebuild` to re-read the CSV files; your flags and proposals are kept.

### Google Colab notes
- Add your key as a Colab secret named `GEMINI_API_KEY` and enable notebook access.
- Use `%run`, not `!python`, because the agent asks for typed input (questions and approvals).

### Terminal
```bash
export GEMINI_API_KEY="your-key-here"     # Windows PowerShell: $env:GEMINI_API_KEY="your-key-here"
python olist_agent.py --data olist
```
If no key is found, the script asks for one with hidden input. **Never put your key in the code or commit it.**

---

## Example questions

**Lookups and summaries**
- Which sellers with at least 50 orders have the highest late-delivery rate, and what are their average reviews?
- Which states have the worst late-delivery rate?
- What were the top 5 product categories by revenue in 2017?
- Show monthly revenue for 2017.
- What data problems does this dataset have?

**Multi-step (follows links)**
- Find orders with a review score of 1, then show their items and sellers.

**Actions** (you will be asked to approve)
- Flag order `<order_id>` for review because the delivery was very late.
- Propose a partial refund for order `<order_id>`.

**Safety checks** (should be refused)
- Propose a refund larger than the order's total.
- Flag the same order twice.

Watch the `tool>` lines in the output: each one is a real lookup in the ontology.

---

## Testing

`python olist_agent.py --selftest` runs 43 checks on a generated sample in Olist's format, with no AI involved. It covers:
- bad or missing files give clear errors;
- search, filters, links and the 25-row cap;
- every summary tool compared against an **independent pandas calculation**;
- rule violations are refused and rejected actions change nothing;
- the original data is untouched by actions;
- the audit log records the expected events, and a rebuild keeps flags and proposals.

These tests use fake data in the same format. The agent has not been formally verified against every detail of the real files, and the live Gemini conversation depends on your key and Google's service availability.

---

## Data

- Dataset: **Brazilian E-Commerce Public Dataset by Olist** (Kaggle). Check the license on its Kaggle page and credit Olist if you reuse or share anything derived from it.
- The data is **historical (about 2016 to 2018)** and anonymized. The agent is told not to treat it as current.
- Money is in **BRL** (Brazilian real).
- **Do not commit the dataset, the database or the audit log** to GitHub. The included `.gitignore` excludes them.

---

## Limitations

- Actions are demonstrations: flags, refund *proposals* and notes in local tables, not a real order system.
- Single user with terminal approval; no roles or permissions.
- The data is a snapshot; there is no live pipeline or refresh schedule.
- Review comments are not loaded, only scores.
- Late-delivery rates for small groups are unreliable, so the tools apply minimum sizes.
- The agent's answers still depend on the language model reading tool results correctly. Check important numbers against the tool output.

## Troubleshooting

| Problem | Fix |
|---|---|
| `Data problem: ... missing columns` | The CSV does not match the expected Olist columns. The message names the file and column. |
| `No database yet and no CSV folder found` | Run with `--data FOLDER` pointing at the unzipped dataset. |
| `401` key rejected | Run `pip install -U google-genai` and restart. Use an AI Studio key, not a Vertex AI express key. |
| `404` model not found | Set `GEMINI_MODEL` to a current model name from AI Studio. |
| `429` quota or rate limit | Wait a minute and retry. |
| `503` model busy | Temporary. The agent retries and falls back to other models (chat memory resets when it switches). |
| Prompt hangs in Colab | Use `%run olist_agent.py`, not `!python`. |

---

## Project files

| File | Purpose |
|---|---|
| `olist_agent.py` | The whole agent: loader, ontology, tools, actions, audit log, Gemini loop, sample generator, self-test |
| `README.md` | This file |
| `.gitignore` | Keeps data, databases and logs out of the repo |

## Notes

This is a learning project. The code was written with AI assistance and used as a reference to study and run an ontology-based agent.

## Key takeaway

> Don't give the AI your database. Give it a clean model of the business, narrow tools, rule-checked actions, your approval, and a log. It stops guessing and starts looking things up.
