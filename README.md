# HealthBot — AI-Powered Patient Education System

LangGraph capstone for MediTech Solutions. A conversational workflow that searches reputable
medical sources with Tavily, summarizes them in patient-friendly language, quizzes the patient,
grades the answer with citations, and then loops for a new topic or exits — resetting state
between topics for privacy.

## Files

| File | Purpose |
| --- | --- |
| `HealthBot.ipynb` | The project notebook — graph, nodes, edges, prompts, and the run cell |
| `docs/HLD.md` | High Level Design — architecture, data flow, design decisions, limitations |
| `docs/LLD.md` | Low Level Design — state schema, node contracts, routing, prompts, verification |
| `requirements.txt` | Pinned dependency list |
| `config.env.example` | Template — copy to `config.env` and add your keys |
| `config.env` | **Your API keys** (git-ignored, never committed) |

## Setup

```bash
pip install uv
uv venv --python 3.11.13
.\.venv\Scripts\Activate
uv pip install -r requirements.txt
```

Then add your keys to `config.env`:

```env
GROQ_API_KEY="gsk_**********"
TAVILY_API_KEY="tvly-********"
```

Free keys: Groq at https://console.groq.com/keys, Tavily at https://app.tavily.com/home
(first 1000 Tavily requests free).

### Provider note

The reference brief specifies the OpenAI API. This build runs OpenAI's open-weight model
**`openai/gpt-oss-120b` on Groq inference**. Nothing else differs — the model still calls the LangChain community Tavily
tool via `bind_tools`, and the graph, state and prompts are provider-independent. To move back to
OpenAI, change two lines in notebook section 2:

```python
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
```

Launch the notebook:

```bash
.\.venv\Scripts\Activate; jupyter notebook HealthBot.ipynb
```

A kernel for this venv is already registered as **Python 3.11 (HealthBot)** — pick it from
*Kernel → Change Kernel* before running. (To re-register it: `.\.venv\Scripts\python.exe -m
ipykernel install --user --name healthbot --display-name "Python 3.11 (HealthBot)"`.)

Then run cells top to bottom. Section 8 is the interactive session.

## Workflow

```
START
  └─ collect_topic          patient types a health topic            [input()]
     └─ search_agent        the model emits a Tavily tool call
        ├─ tavily_tools     ToolNode executes the Tavily search
        └─ collect_results  flatten ToolMessage -> state
           └─ summarize     3-4 paragraph patient-friendly summary (search results only)
              └─ display_summary                                    [print()]
                 └─ readiness_check   "press Enter when ready"      [input()]
                    └─ generate_quiz  one question, summary only
                       └─ present_quiz                              [print()]
                          └─ collect_answer                         [input()]
                             └─ grade_answer   letter grade + justification + citations
                                └─ present_grade                    [print()]
                                   └─ ask_next_action  new / exit   [input()]
                                      ├─ reset_state ─► collect_topic   (loop, state wiped)
                                      └─ end_session ─► END
```

## How each rubric item is met

**LangGraph configuration** — keys loaded from `config.env` with `python-dotenv` and asserted
(section 1). Section 2a makes a live call to both services and prints the model's `tool_calls`,
showing the model invoking Tavily. In the graph, `search_agent` uses `llm.bind_tools([tavily_tool])`
and a `ToolNode` executes the call.

**Summarization** — `summarize` feeds only the flattened Tavily `ToolMessage` content to the model
under `SUMMARY_SYSTEM_PROMPT`, which forbids outside knowledge and requires 3–4 plain-language
paragraphs at a 6th–8th grade reading level.

**Quiz creation** — `generate_quiz` passes only `state["summary"]`; the prompt requires exactly one
question that is fully answerable from the summary alone.

**Grading** — `grade_answer` passes the summary, the question and the patient's answer, and the
prompt fixes the output format: a letter grade, a justification, and direct quotations from the
summary as citations.

**State management** — `HealthBotState` is a `TypedDict` with `messages`
(`Annotated[list, add_messages]`) plus `topic`, `search_results`, `sources`, `summary`,
`quiz_question`, `patient_answer`, `grade_feedback` and `session_active`. Each node returns only the
keys it changes, and downstream nodes read what upstream nodes wrote. Because tool calls, tool
results, the summary, the quiz question and the patient's answer all land in `messages`, the model
has the full conversation available.

**Nodes & edges** — 15 single-responsibility nodes, two conditional edges (tool routing, and
new-topic vs. exit), and a loop back through `reset_state`. `reset_state` blanks every field and
emits a `RemoveMessage` for each existing message so the `add_messages` reducer deletes the previous
topic's history. Section 9 inspects the final state to prove the reset worked: after a two-topic
session, only the most recent topic remains and the message count is one topic's worth.
