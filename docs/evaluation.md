# Evaluation set

A small fixed question set used to check the bot after every change. Each question has a reference: the file the answer must come from, the facts it must contain, and a known trap (a plausible wrong answer).

How each answer is scored:

1. **Retrieval** — was the expected source file among the retrieved chunks (checked in the n8n execution log, not only in the model's "Sources" line)?
2. **Facts** — how many of the expected facts are present?
3. **Hallucinations** — any identifier, field or function that does not exist in the documentation?

Questions are asked in a fresh conversation (new memory session key) so earlier turns do not leak into the answer. The bot is asked in Russian; questions are kept verbatim below.

## Main set

| # | Question | Expected source | Expected facts | Trap |
|---|---|---|---|---|
| 1 | Что делает нода Set Parameters в Корезоиде и какие у неё поля? | `set-parameters-node.md` | `extra`, `extra_type`, `err_node_id` (required), Conditional Setting | an invented `$.now()` function |
| 2 | Какие встроенные функции есть в ноде Set Parameters в Корезоиде? | `set-parameters-built-in-functions.md` | `$.math`, `$.random`, `$.date`, `$.unixtime`, `$.base64_encode`, `$.md5`/`$.md5_hex`, SHA family, `$.map`, `$.filter` | invented functions |
| 3 | Какое минимальное время задержки в ноде Delay в Корезоиде? | `delay-node.md` | 30 s for a static value; a dynamic value is an absolute Unix timestamp and is not subject to the static minimum | "1 second" / "no limit" |
| 4 | Какой type у ноды Call a Process в Корезоиде и что должен делать вызванный процесс? | `call-process-node.md` | `api_rpc` (not `call_process`); the called process must reply (`api_rpc_reply`) on every path or the caller hangs until the timeout | `api_copy` |
| 5 | Что будет в Корезоиде, если у ноды Condition нет ветки по умолчанию? | `condition-node.md` | a final `go` logic is required; without it the error "Condition node has no link" and tasks can get stuck | "it just continues" |
| 6 | Какая сегодня погода? | none (no retrieval needed) | polite refusal: only n8n and Corezoid topics | answering about the weather |

## Held-out set

Added after the prompt had been tuned on the main set, to check that the fix generalizes instead of memorizing one question. The target here is identifier confusion (`api_rpc` / `api_copy` / `api_callback` / `api_get_task`).

| # | Question | Expected source | Expected facts | Trap |
|---|---|---|---|---|
| H1 | Какой type у ноды Waiting for Callback в Корезоиде? | `waiting-for-callback-node.md` | `api_callback`, an auto-generated public callback URL, timeout through a time semaphore | confusing it with `api_rpc` |
| H2 | Как в Корезоиде реализована нода Modify Task, отдельный ли у неё type? | `modify-task-node.md` | no separate type: `api_copy` with `mode: "modify"`, addressed by `ref` | a separate "modify_task" type |
| H3 *(not run yet)* | Какой type у ноды Copy Task в Корезоиде и какие параметры она принимает? | `copy-task-node.md` | `api_copy`, `mode: "create"`, `data`/`data_type`, `group: "all"` when `data` is used; the original task continues | `api_rpc` |
| H4 *(not run yet)* | Какой type у ноды Get from Queue и где лежат данные полученной задачи? | `get-from-queue-node.md` | `api_get_task`; data is placed under `__queue_task_data__`, absent when the queue is empty | data in the task root |

## Results

| Version of the system | Q1 facts | Hallucinations | Sources cited |
|---|---|---|---|
| Model memory only (no retrieval) | 1/4 | yes, a wrong claim about an n8n option default | none |
| Retrieval over a broken index (only 100 of 618 chunks had vectors) | 2/4 | yes, `$.now()` | no |
| Fixed index + grounding prompt | 4/4 | none | yes |

Main set after the final prompt: **6/6**. Held-out set: **2/2** run so far, with one minor omission (H1 did not mention the public callback URL).

The first run of question 4 failed: the model answered `api_copy`. Retrieval had returned the Modify Task chunk (which literally contains `"type": "api_copy"`) instead of the Call a Process chunk, because the correct chunk did not contain the words "Call a Process" (its heading sat in the previous chunk). A prompt rule about search queries fixed it; see the findings in the README.

## Caveats

- Eight questions and single runs: a smoke test, not a benchmark. LLM answers vary between runs; important questions should be repeated several times.
- The prompt rule added for question 4 was tailored to that failure, which is why the held-out questions exist.
- The n8n analogies in the answers are not grounded: only Corezoid documentation is in the knowledge base.
