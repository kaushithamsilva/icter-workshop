# Hands-On Excercixe Code for  ICTer 2026

* Slides can be found at [slides/2026-ICter-Tutorial.pdf](Slides). 
* Please follow along to try out the Hands-on tasks. 
* Notes were made on Nov 03, 2026; if you check later, the instructions for getting API keys, etc., may have changed. 


# Get a free OpenRouter API key

Both hands on sessions call models through OpenRouter, so create a key first.

1. Go to [openrouter.ai](https://openrouter.ai) and sign up or sign in.
2. Open the [API keys page](https://openrouter.ai/settings/keys) and click **Create API Key**.
3. Give the key a name (for example `workshop`), leave any credit limit empty or low, and create it.
4. Copy the key straight away. OpenRouter shows it only once.

You do not need to add credits. The workshop uses free models (`openrouter/free` or
models ending in `:free`), which cost nothing but are rate limited, so a run may
pause or retry. Treat the key like a password: keep it in `.env` (which is
git-ignored), and never paste it into code, chat or a commit.

# Hands on Session 01

A small CLI that sends one task to several LangGraph agents (via OpenRouter) so you
can see how context and tool design change the result.

| Task | Lesson |
|---|---|
| `check-my-balance` | Context and tools change what the agent can do |
| `wrong-tool-description` | A misleading tool description makes the agent misuse the tool |
| `wrong-error-message` | A misleading error message stops the agent from recovering |
| `blocked-account` | The agent asks for payment, a script pays, the agent retries |

## Setup

```bash
uv sync
cp .env.example .env     # then add your OPENROUTER_API_KEY
```

Open `.env` and replace `your-key-here` with the key you created in
[Get a free OpenRouter API key](#get-a-free-openrouter-api-key):

```env
OPENROUTER_API_KEY=your-key-here
```

Model precedence: `--model`, then `OPENROUTER_MODEL` in `.env`, then `openrouter/free`.
The model must support tool calling.

Run any task with `uv run main.py <task>`, optionally with `--scenario <name>` to run a
single scenario and `--model <id>` to choose a model.

## Task: `check-my-balance`

The user asks "Check my balance". The mock user is Alex Perera, account `ACC-1001`,
balance 2450.75 USD (all fake, defined in `scenarios/common.py`). The same question goes to
three agents:

| Scenario | What the agent gets | Expected behaviour |
|---|---|---|
| `no-context` | nothing | Says it has no access to any account |
| `context` | user name and account id in the system prompt | Knows who is asking, still cannot know the balance |
| `tools` | the context plus a mocked `get_balance(account_id)` tool | Calls the tool and answers "2,450.75 USD" |

```bash
uv run main.py check-my-balance                    # all three
uv run main.py check-my-balance --scenario tools   # one
uv run main.py check-my-balance --model openai/gpt-4o
```

## Task: `wrong-tool-description`

Shows that a tool's description is what the model relies on. Same question, same
mock account, but the tools differ:

| Scenario | Tools | Result |
|---|---|---|
| `bad-description` | only `delete_account`, described as "Read the current balance for an account id." | The agent trusts the description, calls it, and the account is deleted |
| `good-description` | `get_balance` plus `delete_account` described honestly as destructive | The agent calls `get_balance`, the account survives |

Each run prints whether the account still exists afterwards.

```bash
uv run main.py wrong-tool-description
```

The bad description should reliably cause the deletion.

## Task: `wrong-error-message`

Shows that a tool's error message is how the agent recovers. The agent only knows the
account id as `1001`, but `get_balance` expects `ACC-1001`. Only the error differs:

| Scenario | Error returned for `1001` | Result |
|---|---|---|
| `bad-error` | "Account not found." | The agent believes the account does not exist and gives up |
| `good-error` | "Invalid account_id '1001'. Expected the format 'ACC-' followed by the customer number, for example 'ACC-1001'." | The agent retries with `ACC-1001` and succeeds |

The output lists every id the tool was called with, so the retry is visible.

```bash
uv run main.py wrong-error-message
```

## Task: `blocked-account`

The user asks to add a data plan, but the account is blocked for an unpaid balance
(25.00 USD, mocked). Scenario `pay-and-retry`:

1. The agent calls `add_data_plan`, gets a "blocked, payment required" error, and asks
   the user to pay.
2. A script stands in for the payment call, outside the agent, and unblocks the account.
3. The user says they have paid. The agent retries the same tool and it succeeds.

The retry works because the agent uses an in-memory LangGraph checkpointer with a fixed
`thread_id`, so the second turn continues the same conversation. Swap the mocked payment
in `pay_and_retry` for a real call when you have one.

```bash
uv run main.py blocked-account
```

## Observability

Each scenario prints its trace with `rich`, step by step:

- the task, then the system and user prompts (each shown once)
- every LLM call, with the model that actually answered, latency and token counts
- assistant replies, tool calls with their arguments, and tool responses
- script steps outside the agent (such as the mocked payment) and a final outcome panel

`loguru` logs the same events at full detail to `logs/agent.log` (git-ignored,
rotated at 1 MB). Warnings and errors also go to the terminal.

Tracing is a LangChain callback handler in `scenarios/observability.py`, so scenarios
contain no print statements.

## Notes

- Models are not deterministic, so any scenario can vary between runs.
- `openrouter/free` routes to different free models, even between calls of one run (the
  trace shows which one answered). It can hit per-minute rate limits
  (a 429 error, just retry) and occasionally returns an odd reply, such as a safety
  label instead of an answer. If that happens, pin a model with `--model`.

## Layout

- `main.py`: CLI, routes a task to its scenarios
- `scenarios/`: one module per task, each with a `QUESTION` and a `SCENARIOS` dict
  - `common.py`: model setup, mock user data, `ask` helper
  - `observability.py`: rich trace handler and loguru setup
  - `check_balance.py`, `wrong_tool_description.py`, `wrong_error_message.py`, `blocked_account.py`
  - `__init__.py`: registers the tasks. To add one, create a module and add it to `TASKS`.

---

# Hands on Session 02

All four Tau tasks use the same request: **“My mobile internet is broken or slow;
I need excellent speed, and I have no Wi-Fi.”** The customer is John Smith, phone
`555-123-2002`.

The support agent manages the account; a separate simulated user operates the
phone. Both roles run automatically. You watch their conversation and check
whether the phone/account state satisfies the task. All telecom changes and
charges are simulated.

### 1. Open the repository and install dependencies

If you already have this checkout, open a terminal in its root directory—the
folder containing `telecom.py`, `showprogress.py` and `pyproject.toml`. Otherwise,
clone the repository and enter the folder:

```bash
git clone https://github.com/kaushithamsilva/icter-workshop.git
cd icter-workshop
```

Install [uv](https://docs.astral.sh/uv/getting-started/installation/) if needed,
then install the locked dependencies:

```bash
uv sync --frozen
```

The repository selects Python 3.12 and includes the required Tau core and data.
You do not need another Tau checkout or to activate a virtual environment.
Run every command below from this repository root.

### 2. Choose the model

This session uses the `OPENROUTER_API_KEY` already set in `.env` during Session 01
setup.

Use `nvidia/nemotron-3.5-lightning:free` for both the support agent and simulated
user. The live-run commands below select it explicitly with `--model`. It is also
the telecom runner's default, so omitting `--model` selects the same model.
`OPENROUTER_MODEL` in `.env` controls the earlier `main.py` examples; use `--model`
to choose a model for `telecom.py`.

The telecom runner accepts only free models, checks their catalog prices and
sets zero-price provider limits.

### 3. Check that the four task environments load

```bash
uv run telecom.py validate
```

Expected output:

```text
easy: solvable; 4 native environment checks
hard-user: solvable; 4 native environment checks
hard-env: solvable; 7 native environment checks
hard-both: solvable; 7 native environment checks
```

This runs offline and confirms that each task can be solved. It does not run the
agent or prove that the agent will pass.

### 4. Run task 1: easy user and easy environment

In **Terminal A**, run:

```bash
uv run telecom.py easy --model nvidia/nemotron-3.5-lightning:free
```

The customer cooperates, but airplane mode is on and the network preference is
stuck on 2G. The task passes only when all four checks succeed:

- Mobile data works.
- Download speed is at least 200 Mbps and rated **Excellent**.
- Airplane mode is off.
- Cellular service is connected.

Wait for the `Trace: runs/...-easy.jsonl` line. All four run commands show the
`agent` and `user` conversation alongside tool names, arguments and results as
the actions finish. The terminal also shows which phone tools the agent offers
to the customer. No extra flag is needed; add `--no-tools` if you want to hide
tool details. You do not need to type user replies. Keep this terminal running;
use step 5 if you want a separate viewer.

All four commands run with **no conversation-turn limit** by default. They can
continue past turn 12 until the customer stops, leaves or transfers. Press
`Ctrl+C` in the run terminal to stop and save an interrupted run. To set a limit
for a particular run, add `--max-turns 20`. The separate tool-loop limit still
ends a run that gets stuck making tool calls within one speaker's turn.

At the end, read the overall `PASS`, `FAIL` or `ERROR` and the individual check
results. The saved baseline for this task is **PASS, 4/4**. Live model responses
can vary, so a new run is not guaranteed to match the recording.

### 5. Watch the live conversation in a second terminal

After Terminal A prints its `Trace:` line, open **Terminal B** in the same
repository folder and run:

```bash
uv run showprogress.py --follow
```

This displays the latest run's conversation so far, then waits for new messages.
It does not start another task or make model calls. Messages appear when each
turn finishes; this is not token-by-token streaming.

To inspect the account/phone actions as well, press `Ctrl+C` in Terminal B and
restart the viewer with:

```bash
uv run showprogress.py --follow --tools
```

`--tools` adds tool names, arguments, results and the phone tools offered to the
customer. Stopping the viewer does not stop the run in Terminal A.

**Run one live task at a time.** The viewer follows one trace and exits at its
final result. For each new task below, wait for its new `Trace:` line, then run
`uv run showprogress.py --follow` again in Terminal B. A viewer already watching
an older file does not switch to the new task automatically.

### 6. Run task 2: hard user and easy environment

After task 1 finishes, run this in Terminal A:

```bash
uv run telecom.py hard-user --model nvidia/nemotron-3.5-lightning:free
```

The phone has the same two faults and the same **four pass checks** as task 1.
The user is anxious, gives short replies and refuses requests containing several
phone steps. Repeated overload can make the user leave before repairs finish.

After the new `Trace:` line appears, watch it in Terminal B:

```bash
uv run showprogress.py --follow
```

Read how the agent responds when the user asks for one clear step. The saved
baseline is **FAIL, 0/4**: the user leaves after repeated overloaded requests.

### 7. Run task 3: easy user and hard environment

After task 2 finishes, run this in Terminal A:

```bash
uv run telecom.py hard-env --model nvidia/nemotron-3.5-lightning:free
```

The user cooperates. In addition to airplane mode and 2G preference, mobile data
is disabled, Data Saver is on, a faulty VPN is connected and the data allowance
has been exceeded. All four easy checks must pass, plus these three:

- Data Saver is off.
- Data usage no longer exceeds the available allowance.
- Exactly **2 GB** has been added to account `C1001`, line `L1002`.

That makes **seven checks**. The Excellent-speed requirement also catches the
unresolved VPN or slow network mode. The user can approve a 2 GB top-up but does
not want a plan change.

After the new `Trace:` line appears, watch the conversation and actions in
Terminal B:

```bash
uv run showprogress.py --follow --tools
```

Check whether the agent completes every repair and adds the correct amount of
data. The saved baseline is **FAIL, 5/7**: repeated top-ups add 296 GB instead of
2 GB, the speed problem remains, and the tool-step limit stops the loop.

### 8. Run task 4: hard user and hard environment

After task 3 finishes, run this in Terminal A:

```bash
uv run telecom.py hard-both --model nvidia/nemotron-3.5-lightning:free
```

This combines task 2's difficult user with task 3's six faults. Passing requires
the same **seven checks** as task 3.

After the new `Trace:` line appears, watch it in Terminal B:

```bash
uv run showprogress.py --follow --tools
```

Check whether the agent keeps the user cooperating while completing all repairs.
The saved baseline is **FAIL, 0/7**: the user leaves before the faults are fixed.

To run all four tasks sequentially instead of starting them individually, use:

```bash
uv run telecom.py all --model nvidia/nemotron-3.5-lightning:free
```

Terminal A shows every conversation and its tool activity automatically. A
separate viewer still needs restarting for each task. The `all` command stops if
a provider/protocol error occurs. Each task starts with fresh state and no
conversation-turn cap by default; a failure does not carry state into the next
task. Add `--max-turns N` to apply an optional cap to each task.

### 9. Replay conversations and compare results

After a live run finishes, replay its latest conversation with:

```bash
uv run showprogress.py
```

To see the four recorded baselines, run any of these commands. They need the
installed dependencies but **no API key or model calls**:

```bash
uv run showprogress.py examples/easy.jsonl
uv run showprogress.py examples/hard-user.jsonl
uv run showprogress.py examples/hard-env.jsonl
uv run showprogress.py examples/hard-both.jsonl
```

For a paced replay with tool activity, use:

```bash
uv run showprogress.py examples/easy.jsonl --tools --delay 1
```

To replay one of your older runs, replace `examples/easy.jsonl` with the exact
`runs/...jsonl` path printed when that run started. Each run also saves a
`.result.json` file containing the checks and a `.config.json` task/prompt
snapshot beside its trace.

Recorded live results from **30 October 2026**:

| Recording | Result (checks passed) | Model calls | Elapsed |
|---|---|---:|---:|
| [Easy](examples/easy.jsonl) | [**PASS, 4/4**](examples/easy.result.json) | 20 | 101.25 s |
| [Hard user](examples/hard-user.jsonl) | [**FAIL, 0/4**](examples/hard-user.result.json) | 8 | 37.09 s |
| [Hard environment](examples/hard-env.jsonl) | [**FAIL, 5/7**](examples/hard-env.result.json) | 32 | 231.92 s |
| [Both hard](examples/hard-both.jsonl) | [**FAIL, 0/7**](examples/hard-both.result.json) | 7 | 175.08 s |

All four used the same baseline agent prompt and the free NVIDIA model above.
Calls include both roles; all 67 selected replies reported **$0 API cost**.
These selected recordings are not success-rate estimates. The
[attempt index](examples/manifest.json) retains all 17 development attempts,
including errors excluded from scored failures.

Additional live checks from **2 November 2026** are saved separately in
[runtime validation](examples/runtime-validation/manifest.json):

| Task | Result | How the run ended |
|---|---|---|
| Easy | PASS, 4/4 | Explicit four-turn smoke-test cap. |
| Hard user | PASS, 4/4 | Customer stopped after nine turns; no conversation cap. |
| Hard environment | FAIL, 6/7 | Tool-step limit after repeated refuels; exact 2 GB check failed. |
| Both hard | ERROR; 5/7 diagnostic checks true | Recovery exhausted after 13 completed turns; no conversation cap. |

These runs tested successive runtime fixes, so they are not a controlled
comparison. Their 141 model replies all reported $0. The combined hard run still
hit a protocol error; its checks are diagnostic, not a scored failure. The
original four baseline recordings remain unchanged. To replay the uncapped
hard-user run with tool details:

```bash
uv run showprogress.py examples/runtime-validation/20261102T135949684278Z-hard-user.jsonl --tools
```

### 10. Verify the recordings offline

```bash
uv run python scripts/verify_examples.py
uv run python -m unittest discover -s tests -v
```

The first command replays the recorded native tool calls and checks their outputs
and final grades without a model. The second runs 46 offline tests, including
both speakers' identities, empty/truncated reply recovery, tool ownership and
early-exit checks, and a conversation continuing beyond turn 12.

### If a command does not work

| What you see | What to do |
|---|---|
| `uv: command not found` | Install uv using the link in step 1, reopen the terminal, then run `uv sync --frozen`. |
| `telecom.py` or `showprogress.py` cannot be found | Open the repository root, the folder containing `pyproject.toml`, as described in step 1. |
| Missing or invalid API key | Check `OPENROUTER_API_KEY` in the repository's `.env`, save it, and rerun the task. |
| `No run yet`, or the viewer shows an older task | Start a task in Terminal A and wait for its new `Trace:` line before starting the viewer again. |
| `HTTP 429`, a provider outage, or a pause between replies | Keep the run open while a retry is announced. If it ends with `ERROR`, rerun later or replay the saved examples from step 9. |
| The configured free model is unavailable | Replay the saved examples, or explicitly select another available free tool-calling model with `--model`. Paid models are rejected. |
| `Customer role/protocol repair` or `Support role repair` | Read the displayed reason. The runner retries an invalid draft before delivery. Customer retries keep the observed phone results but remove old function-call syntax. Rejected drafts remain in the JSONL trace; three unsuccessful attempts end as `ERROR`. |
| `empty model reply` or `output token limit reached; retry ...` | Keep the run open. The runner retries only that model response, increasing the output limit from 2,048 to 4,096 and then 8,192 when truncated. Completed actions are not replayed. Three unsuccessful response attempts end as `ERROR`. |
| The conversation ends with `FAIL` | Inspect the failed state checks and the tool details shown in the run terminal. For a saved trace, replay with `showprogress.py <trace-path> --tools`. A task failure is different from an API/protocol `ERROR`. |

### About these tasks

This uses [Sierra τ²-bench](https://github.com/sierra-research/tau2-bench)'s native
telecom tools, state and assertions with a custom LangGraph runner. It is not an
official benchmark score. The shared opening and conditional departure are
scripted. The hard user combines Tau's Hard persona with a workshop rule: two
requests offering multiple distinct phone tools trigger departure unless a
single-tool request resets patience between them.

In new runs, **John Smith is the customer** and the other speaker is the support
representative. Both models receive role reminders after tool results. The runner
checks customer replies and support's `ask_customer` messages before delivery,
including attempts by support to introduce itself as John. Customer wording
repairs cannot execute more phone tools; support wording repairs must retain the
same tool calls and requested phone checks. The persona and environment checks
stay the same. Saved examples are historical
recordings and may contain the old role mistakes. Start a new run using steps 4–8
to see the correction; replaying an old trace does not change its dialogue.

All four tasks are solvable. The original selected baselines contain one pass
and three failures; fresh runs can differ, as the later hard-user pass shows.
Runtime guards repair role/protocol errors; agent strategy improvements remain
a later exercise.
See [instructor notes](docs/telecom-details.md) for the source task IDs, exact user
rule and implementation details.
