# 🏪 Retail Ontology Agent

A small, beginner-friendly project that shows how an **ontology** makes an AI agent more reliable. The agent answers retail operations questions (low stock, supplier lead times, markdowns) by querying a structured model of the business instead of guessing, and it can only change data through validated, human-approved actions.

Built with Python and the Gemini API. The idea is inspired by the ontology layer in Palantir Foundry/AIP, scaled down so you can read all of it in one sitting.

---

## Why an ontology?

A plain chatbot has no access to your real data, so it may invent numbers. An ontology gives the agent a **map of the business**:

| Concept | Meaning | Example in this project |
|---|---|---|
| **Object** | A "thing" in the business | Product, Store, Supplier, Inventory |
| **Property** | A fact about an object | `quantity`, `reorder_point`, `price` |
| **Link** | How objects relate | Product → Supplier, Inventory → Store |
| **Action** | A controlled, rule-checked change | `transfer_stock`, `create_purchase_order` |
| **Function** | Reusable logic on objects | `find_low_stock()` |

The agent never touches the data directly. It asks questions through narrow **tools** and acts only through **actions**.

---

## Architecture

```
 You ──question──▶  Gemini model (agent)
                        │  decides which tool to call
                        ▼
        ┌──────────────── TOOLS ────────────────┐
        │ search_objects      get_linked_objects │
        │ find_low_stock      execute_action     │
        └───────────────────┬───────────────────┘
                            ▼
        ┌────────────── ONTOLOGY ───────────────┐
        │ Objects + Properties   (DB)            │
        │ Links                  (LINKS)         │
        │ Actions with rules     (ACTIONS)       │
        └───────────────────┬───────────────────┘
                            ▼
                 Human approval (y/n)
```

---

## The retail ontology

**Object types:** `Supplier`, `Product`, `Store`, `Inventory`, `PurchaseOrder`

**Links**

| From | Link | To |
|---|---|---|
| Product | `supplier` | Supplier |
| Product | `inventory` | Inventory (many) |
| Supplier | `products` | Product (many) |
| Store | `inventory` | Inventory (many) |
| Inventory | `product` / `store` | Product / Store |
| PurchaseOrder | `product` / `supplier` | Product / Supplier |

**Actions** (all require human approval)

| Action | Rule enforced in code |
|---|---|
| `create_purchase_order` | Quantity must be 1 to 1000 |
| `transfer_stock` | Product must exist in both stores; cannot move more than the source holds |
| `apply_markdown` | Discount must be 5% to 50% |

Rules live in the code, not in the prompt, so the model cannot talk its way around them.

---

## Getting started

### Requirements
- Python 3.9+
- A Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) (keys may start with `AQ.` or `AIza`)

### Install
```bash
pip install -U google-genai
```

### Run the self-test (no AI, no key needed)
```bash
python retail_agent.py --selftest
```
This exercises the ontology, links, actions, approvals and rule violations directly.

### Run the agent
```bash
export GEMINI_API_KEY="your-key-here"     # Windows PowerShell: $env:GEMINI_API_KEY="your-key-here"
python retail_agent.py
```
If no key is found, the script prompts for one (input hidden).

### Google Colab
1. Upload `Agent.ipynb`.
2. Add your key as a Colab secret named `GEMINI_API_KEY` and enable notebook access.

## Example questions

**Lookups**
- Which products are low on stock?
- Who supplies the Wireless Earbuds, and how long does delivery take?

**Reasoning**
- Which products are low on stock, and what should we do?
- Which items are overstocked? Suggest a markdown.

**Actions** (you will be asked to approve)
- Move 30 units of rice from the Mall to Downtown.
- Order 50 Phone Chargers.
- Mark down Olive Oil by 20%.

**Safety checks** (should be refused)
- Mark down Olive Oil by 80%.
- Move 500 units of rice from the Mall to Downtown.

Watch the `tool>` lines in the output: each one is a real lookup in the ontology.

---

## How the code is organised

| Section in `retail_agent.py` | Purpose |
|---|---|
| **1. Data** (`DB`) | In-memory objects and properties |
| **2. Ontology** (`LINKS`, `ACTIONS`) | Link types and rule-checked actions |
| **3. Tools** | The four functions the model may call |
| **4. System prompt** (`SYSTEM`) | Tells the model the ontology and its rules |
| **5. Agent** (`get_api_key`, `pick_models`, `ask`, `agent`) | Key handling, model selection, retry and fallback, chat loop |
| **6. Self-test** (`selftest`) | Tests everything without an LLM |

Highlights:
- **Errors are returned, not raised.** A bad link returns the list of valid links, so the model can correct itself.
- **Automatic tool calling.** Plain Python functions are passed to Gemini, which runs the think → call tool → read result loop.
- **Resilience.** Temporary Google errors (429, 500, 502, 503, 504) trigger retries, then fallback to other models. If a model switch happens, chat memory resets.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `401` key rejected | Run `pip install -U google-genai` and restart. Make sure the key is from AI Studio, not a Vertex AI express key. |
| `404` model not found | Set `GEMINI_MODEL` to a current model name from AI Studio. |
| `429` quota or rate limit | Wait a minute and retry. |
| `503` model busy | Temporary on Google's side. The agent retries and falls back automatically. |
| Input prompt hangs in Colab | Use `%run retail_agent.py`, not `!python`. |

---

## Limitations

- Data is **in memory**: changes reset when the script restarts.
- Demo data only (4 products, 3 stores). Do not use real customer data with a free API tier without checking its data terms.
- One action at a time; no user roles or audit log yet.

## Roadmap ideas

- Load products and inventory from a CSV file
- Add an audit log of questions, tool calls and approvals
- Add a Sales object and a demand forecast function

---

## Key takeaway

> Don't give the AI your database. Give it narrow tools and rule-checked actions. It stops guessing and starts looking things up.

## License

MIT. Add a `LICENSE` file if you publish this.
