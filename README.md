# FoodHub Customer-Support Chatbot

A chatbot that answers customers' questions about their FoodHub order from the order database, with input and output guardrails and escalation to a human agent.

## Files

| File | Purpose |
|---|---|
| `FoodHub_Chatbot_FullCode_Notebook.ipynb` | Cleaned notebook (no saved outputs) |
| `requirements.txt` | Python dependencies |
| `make_sample_db.py` | Builds a 10-row sample `customer_orders.db` for offline testing (not needed if you have the real one) |
| `config.json` | **Yours, not included.** `{"OPENAI_API_KEY": "...", "OPENAI_API_BASE": "..."}` |
| `customer_orders.db` | **Yours, not included.** SQLite database with an `orders` table |

## How to run

1. `pip install -r requirements.txt`
2. Put `config.json` and `customer_orders.db` next to the notebook (or set `FOODHUB_CONFIG` / `FOODHUB_DB`, or edit the Configuration cell). On Colab, the files are read from `Colab Notebooks` in Drive. If there is no config file, the key is read from the `OPENAI_API_KEY` environment variable.
3. Run all cells. Test conversations run from fixed scripts, so nothing needs typing. For a live chat, run `chatagent()`.

Keep `config.json` out of version control and shared folders.

## How it works

1. **Verify**: the customer gives an order ID and customer ID. The order is fetched with a parameterised, read-only SQL query; the bot never sees any other order.
2. **Input guardrail**: an LLM classifies each message as 0 escalate, 1 exit, 2 order query, 3 off-topic or adversarial, 4 acknowledgement. Anything unparseable stops the chat with a technical-issue message.
3. **Answer**: one LLM call that sees only that order record and the recent chat, with rules not to invent facts or policies.
4. **Output guardrail**: blocks any reply naming an order or customer ID other than the customer's own, then an LLM `SAFE`/`BLOCK` check (anything but `SAFE` blocks).
5. **Escalation**: `escalate_to_human` records the case in `ESCALATION_LOG`. It is a placeholder for a ticketing or live-chat integration.

## What was changed from the original notebook

**Security and correctness**
- **Order lookup was injectable.** The typed order ID was pasted into a sentence sent to the SQL agent, so a crafted ID or instruction could pull other orders. It now uses format validation plus a parameterised query on a read-only connection, and requires a matching customer ID. (Test cells show malformed IDs, `OR 1=1`, `DROP TABLE` text, a wrong customer ID and an unknown order are all refused.)
- **Output guardrail failed open**: any reply other than `BLOCK` was shown. It now fails closed, and a deterministic check blocks foreign order/customer IDs.
- **Input guardrail failure handling**: the digit was extracted by stripping non-digits, so a reply like "maybe 2?" could be misread. Now only an exact `0` to `4` is accepted.
- **Invented policy**: the answer for "I want to cancel" said the order "cannot be canceled" though no policy was ever provided. The prompt now forbids stating policies and says to give the status and offer a human agent.
- **Dropped rows**: the original showed an LLM-written summary table with 10 rows although its own count query returned 20 unique order IDs. The full table is now read with pandas.
- **Redundant LLM calls**: the two "tools" each called the LLM and `answer_tool` re-ran `order_query_tool`. Replaced with one grounded answer call.
- The database is opened read-only for the SQL agent too.

**Code health**
- Removed deprecated APIs (`langchain.chat_models.ChatOpenAI`, `llm.predict`, `initialize_agent`, `agent.run`) in favour of `langchain_openai` and `invoke`.
- Conversation logic is in `chat_turn()`, separate from `input()`, so it can be tested.
- Chat history is a list, with a length cap; messages over 500 characters are refused; API errors end the chat gracefully.
- The input-guardrail description listed 4 codes while the prompt used 5; both now say 5.
- Colab-only code is optional; paths are configurable; the exact-version `pip install` was replaced by `requirements.txt`.
- Conclusions no longer assert results that are not measured; they describe the design and what to check when re-running.

**Behaviour change to be aware of**: customers must now supply a customer ID. Set `REQUIRE_CUSTOMER_ID = False` to go back to order ID only (not recommended).

## What was and was not verified

I could not call your LLM endpoint from this environment, so **the notebook has not been run against the real model**.

What was run: all non-LLM cells and the whole chat flow, against your real `customer_orders.db` (20 orders, 20 unique IDs, no duplicates; missing times are true NULLs, which the notebook shows as "not available"), with a stub standing in for the model. That checked the database access, the security refusals, the routing of codes 0 to 4, the handling of an invalid guardrail reply, blocked output, over-long input, and escalation logging. It did not test how well the real model classifies or answers; that needs a run with your credentials.

## Notes on the original run

Saved outputs in the original notebook showed the four test queries behaving as intended (adversarial query deflected, angry customer escalated, status answers for order O12491). The database holds 20 orders; the original LLM-written table showed only 10.

## Ideas not applied

A labelled evaluation set for the guardrails, a real ticketing hand-off, authenticated sessions instead of typed IDs, and explicit tools for cancellations or address changes (these need the business's own rules).
