# 🧠 Ontology + AI Agents: Two Worked Examples

A learning project that shows how an **ontology** makes an AI agent more reliable. Two agents are built with Python and the Gemini API, each answering business questions by querying a structured model of the data instead of guessing, and each able to change data only through rule-checked, human-approved actions.

| | Agent 1: Retail | Agent 2: Olist e-commerce |
|---|---|---|
| Notebook | `Agent.ipynb` | `Agent2.ipynb` |
| Details | [README_Agent.md](README_Agent.md) | [README_Agent2.md](README_Agent2.md) |
| Open in Colab | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/vijayalakshmish/Agent/blob/main/Agent.ipynb) | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/vijayalakshmish/Agent/blob/main/Agent2.ipynb) |

Palantir Foundry/AIP is used **for reference only** to understand the ontology concept. Nothing here is built on AIP.

---

## The idea in two minutes

A database stores facts. An ontology adds **meaning, connections and behavior** on top of them, so an AI can look things up instead of guessing.

| Building block | Meaning | Retail example | Olist example |
|---|---|---|---|
| **Object** | A real-world thing | Product, Store | Order, Seller |
| **Property** | A fact about it | `quantity`, `price` | `review_score`, `is_late` |
| **Link** | A named relationship | Product → Supplier | Order → items → Seller |
| **Action** | A rule-checked change | `transfer_stock` | `propose_refund` |
| **Function** | Reusable logic on objects | `find_low_stock()` | `seller_performance()` |

**How the agent benefits**

1. **Grounded answers.** Numbers come from tool calls on real data, not from the model's memory.
2. **Multi-step reasoning.** Links let it follow a chain, such as late order → seller → reviews.
3. **Safe actions.** Rules live in code, so the model cannot talk its way around them, and a person approves every change.
4. **Manageable scale.** Narrow tools and result limits keep the model from drowning in data.
5. **Traceability.** Tool calls and results can be inspected (Agent 2 also keeps an audit log).

```
You ──question──▶ AI agent ──▶ narrow TOOLS ──▶ ONTOLOGY (objects, links, rules)
                                                   │
                              changes only via ACTIONS ──▶ human approval
```

---

## What is in this repository

| Item | What it is |
|---|---|
| `concept diagrams/` | Diagrams explaining the core ideas used in both agents |
| `Agent.ipynb` | Agent 1: retail ontology agent (hand-made data) |
| `Agent2.ipynb` | Agent 2: Olist e-commerce ontology agent (real, multi-table dataset) |
| `README_Agent.md` | Detailed notes for Agent 1 |
| `README_Agent2.md` | Detailed notes for Agent 2 |
| `README.md` | This overview |

---

## The two agents compared

| | Agent 1: Retail | Agent 2: Olist |
|---|---|---|
| **Data** | 4 products, 3 stores, built into the code | Olist Brazilian marketplace, about 100,000 orders, loaded from CSV files |
| **Storage** | In-memory Python dictionaries | SQLite database |
| **Object types** | Supplier, Product, Store, Inventory, PurchaseOrder | Customer, Order, OrderItem, Product, Seller, Review, Payment |
| **Tools** | 4: search, follow links, find low stock, execute action | 7: search, follow links, delivery stats, seller performance, sales summary, data-quality report, execute action |
| **Actions** (all need approval) | Purchase order, stock transfer, markdown | Flag order, refund *proposal*, seller note |
| **Examples of rules** | Order 1 to 1000 units; markdown 5% to 50% | Refund cannot exceed amount paid; an order cannot be flagged twice |
| **Dry-run validation** | Rules checked when the action runs | Rules checked first, then approval, then applied in one transaction |
| **Computed properties** | `find_low_stock()` function | `is_late`, `delivery_days` calculated by a database view |
| **Audit log** | No | Yes (`audit_log.jsonl`) |
| **Changes persist?** | No, memory only | In separate action tables; the original data is never edited |

**What Agent 2 adds over Agent 1:** a real dataset with real links between tables, summary tools instead of raw rows, a stricter action pipeline, and an audit log. Agent 2 is Agent 1's pattern applied at a more realistic scale.

---

## Agent 1: Retail

Answers retail operations questions: which items are low on stock, who supplies them, and what to do about it.

- **Ontology:** five object types linked by named links (for example Product → Supplier, Inventory → Store). `Inventory` is its own object because stock is a fact about a product in a store.
- **Example questions:** "Which products are low on stock, and what should we do?" · "Move 30 units of rice from the Mall to Downtown." · "Mark down Olive Oil by 80%." (refused)
- **Details:** [README_Agent.md](README_Agent.md)

## Agent 2: Olist e-commerce

Answers questions about orders, deliveries, sellers, reviews and sales on a real marketplace dataset.

- **Ontology:** seven object types and twelve links. Fields, operators and tables are checked against whitelists and values are passed as parameters, so the model cannot write its own SQL. Tools return capped results (25 rows) or summaries.
- **Example questions:** "Which sellers with at least 50 orders have the highest late-delivery rate?" · "What were the top categories by revenue in 2017?" · "Flag order `<id>` for review." · "Propose a refund larger than the order total." (refused)
- **Details:** [README_Agent2.md](README_Agent2.md)

---

## Getting started

### Requirements
- A Google account and a free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) (keys may start with `AQ.` or `AIza`)
- Python 3.10+ if you run outside Colab

### In Google Colab (easiest)
1. Click an **Open In Colab** badge above.
2. Click the key icon, add a secret named `GEMINI_API_KEY`, and turn on notebook access.
3. Run the cells in order and ask questions. Type `quit` to stop.

### Agent 2 data
Agent 2 needs the Olist dataset from Kaggle: <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce>

```python
!pip install -U google-genai pandas kagglehub
import kagglehub, shutil
path = kagglehub.dataset_download("olistbr/brazilian-ecommerce")
shutil.copytree(path, "olist", dirs_exist_ok=True)
```
If `kagglehub` asks for Kaggle credentials, follow its prompt. See [README_Agent2.md](README_Agent2.md) for the full steps.

### Common problems

| Problem | Fix |
|---|---|
| `401` key rejected | Run `pip install -U google-genai` and restart. Use an AI Studio key, not a Vertex AI express key. |
| `404` model not found | Set `GEMINI_MODEL` to a current model name from AI Studio. |
| `429` quota or rate limit | Wait a minute and retry. |
| `503` model busy | Temporary. The agents retry and fall back to other models (chat memory resets when they switch). |

---

## Testing

Both agents include a built-in self-test that runs **without any AI or API key**. Agent 2's runs 43 checks on a generated sample in the Olist format, comparing its tools against independent pandas calculations and confirming that rule violations are refused and rejected actions change nothing. See each agent's README for how to run it.

Live conversations with Gemini depend on your key and Google's service availability, so check important numbers against the `tool>` output.

---

## Data, keys and privacy

- **Never commit your API key.** Use Colab secrets or environment variables.
- **Olist data:** check the license on the Kaggle page and credit Olist if you reuse or share anything derived from it. The data is historical (about 2016 to 2018), anonymized, and amounts are in BRL.
- **Do not commit** the dataset, SQLite databases or audit logs.
- Do not send real customer data to a free API tier without checking its data terms.

---

## Limitations

- Actions are demonstrations. Agent 1 changes in-memory data; Agent 2 writes flags, refund *proposals* and notes to local tables. Neither connects to a real order system.
- Single user with terminal approval; no roles or permissions.
- Data is a snapshot with no live pipeline.
- The model can still misread tool results, so verify key numbers.

## Ideas for next steps

- Load Agent 1's data from CSV and add an audit log
- A browser screen with Approve and Reject buttons
- Add a graph-database version (for example Neo4j) of either ontology
- Add roles and permissions so the agent acts only as the person using it

---

## Notes

This is a learning project. The code was written with AI assistance and used as a reference to study, run and understand ontology-based agents.

## Key takeaway

> Don't give the AI your database. Give it a clean model of the business, narrow tools, rule-checked actions, your approval, and a log. It stops guessing and starts looking things up.
