# OpenScript

[![CI](https://github.com/openscript-ai/openscript/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/openscript-ai/openscript/actions/workflows/ci.yml)
[![PyPI](https://img.shields.io/pypi/v/openscript-sdk?label=pypi)](https://pypi.org/project/openscript-sdk/)
[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-blue)](https://github.com/openscript-ai/openscript/blob/master/LICENSE)
[![Live demo](https://img.shields.io/badge/demo-live%20in%20your%20browser-22c55e)](https://openscript-demo.vercel.app/)

**Irreversible actions wait for a person, even when the model has been fooled.**

## Project overview

**What it does.** OpenScript sits between an AI agent and the tools it
calls. It refuses calls outside rules you write, holds the ones that cannot
be undone until a person approves that exact call, masks secrets and PII,
and records every decision.
None of it depends on recognising an attack, so it still holds when the model
is fooled, wrong, or swapped for another.

It is for anyone building an agent whose tools have real effects, such as
payments, email or a shell, in Python with any framework. It is version
0.1.0 and alpha: the API may still change.

**The problem it addresses.** A tool-using agent can be talked into acting
by text it reads: a web page, an email, a payment reference. Current models
resist most of these attacks, but a weaker model follows them, and pattern
detectors catch 2 to 7% of attacks they were not written for. So neither
the model nor detection is a boundary. An agent whose tools pay, send or
delete needs one that holds when the model has been fooled.

**[Try the live demo →](https://openscript-demo.vercel.app/)** The package
runs in your browser tab, compiled to WebAssembly, with no backend.

## Safety approach

How it helps prevent unauthorized or irreversible AI actions. Nothing here
depends on recognising an attack: the attacker is assumed to be the model
itself, from the first turn.

```
agent ── calls a tool ──► SecureTool ── before: tool rules · session rule ──► refuse / hold for a person
                              │
                              ▼  the call that was checked, and nothing else
                         effect side ── a person's token? ──► apply, and write it in the ledger
                              │
                              ▼
                          after: mask secrets and PII · record
```

**Against unauthorized actions**, rules the agent cannot widen:

- **Tool rules** limit which tools, arguments, recipients and amounts are
  allowed, in YAML. With `default_deny`, a tool with no rule is refused.
- **It fails closed.** A policy that raises, or an approval store that
  cannot be reached, blocks the call.
- **Every decision is recorded,** with secrets and PII masked.

**Against irreversible actions**, a person, for that exact call, checked
twice:

- **Each tool is classed** as read, reversible or irreversible. Irreversible
  means something leaves the environment, or no tool can put back what
  changed.
- **The session rule** remembers when a session has read something a
  stranger wrote, and then holds the calls you list as irreversible.
- **An approval** is single-use, expires in an hour, and is bound to the
  tool, its exact arguments and the session.
- **The effect side** ([`evals/ledger.py`](https://github.com/openscript-ai/openscript/blob/master/evals/ledger.py)) stands
  in front of each irreversible tool and applies a call only with a token
  minted when a person approves it, under a key the agent never holds. It
  is the boundary, because it is the only part the agent's process does not
  own. It is a reference implementation, standard library only.

## Supporting evidence

### Demo

- **[The live demo](https://openscript-demo.vercel.app/)** runs the package
  in your browser.
- **[A recorded walkthrough](https://github.com/openscript-ai/openscript/raw/master/docs/demo.mp4)**, under a minute: a
  tool rule refuses a form post that a web page provoked, a call is held
  for a team lead and approved once, and an API key is masked before the
  model is given it.

### Test results

Measured on [AgentDojo](https://github.com/ethz-spylab/agentdojo)'s banking
and Slack suites, with Qwen2.5-7B-Instruct, a model that follows prompt
injections. The protocol was frozen before the run.

| The run | |
|---|---|
| Hardware | 1 NVIDIA RTX 4090 on RunPod, Secure Cloud, EU-RO-1 |
| Serving | vLLM 0.10.2, bf16, temperature 0 |
| Size | 1,430 cases, 5,382 requests, 9.8M input and 0.63M output tokens |
| Errors | 0 |
| Date | 2026-11-04 |

| Configuration | Attacks that succeeded | Attacker's irreversible actions that took effect |
|---|---|---|
| No protection | 58 of 249 | 108 |
| Session rule: hold irreversible calls after an untrusted read | 6 of 249 | 10 |
| Ask a person on every irreversible call | **0 of 249** | **0** |
| No policy at all, effect side enforcing | **0 of 249** | **0** |

- **The effect side is the boundary.** A ledger in front of each irreversible
  tool applies a call only with a token minted when a person approves it. A
  call that goes round the wrapper changes nothing.
- **Safety has a price, and it is published.** In the run above, asking on
  every irreversible call meant a person on 23 of the 37 ordinary tasks. In
  the no-model replay of all four suites it is 59 of 97. The session rule
  asks less and misses the attacks that arrive through Slack's channel
  listing.
- **It was attacked before it was trusted.** A [red team](https://github.com/openscript-ai/openscript/blob/master/docs/red-team.md)
  searched for bypasses and published 7 findings, each committed as a
  failing test before its fix.

Every number, with its date, command and caveats, is in
[`docs/metrics.md`](https://github.com/openscript-ai/openscript/blob/master/docs/metrics.md). The claim and what it does not cover
are in [the threat model](https://github.com/openscript-ai/openscript/blob/master/docs/threat-model.md).

To rerun the tests and the no-model replay, free and with no API key:

```bash
make install && .venv/bin/pytest          # 909 pass, 16 skipped
make agentdojo-replay
```

### Real-world usage: Countersign

[Countersign](https://github.com/saintparish4/countersign) is a
spending-control gateway between a Claude support agent and PayPal refunds.
It installs `openscript-sdk` 0.1.0 from PyPI and uses it unmodified.

- **What OpenScript does there:** it checks the shape of each refund request
  (`ToolFirewallPolicy`), runs the policies fail-closed around the refund
  tool (`SecureTool`), and binds each approval to one exact refund. A refund
  above the merchant's auto-limit is held, and nothing is sent until a
  member of staff approves it.
- **What it was run against:** the PayPal sandbox, on 2026-11-12, with the
  gateway and the agent in separate containers. An 80.00 refund was held
  and sent only after approval, and a refund naming another customer's
  order was denied. Its README lists the refund ids.

It is by the same author, sandbox only and unaudited, so it shows the
package carrying a real payment API's irreversible action, not a production
deployment.

### Documentation

- [`docs/threat-model.md`](https://github.com/openscript-ai/openscript/blob/master/docs/threat-model.md): the attacker, the claim, and
  every path to an effect, marked mediated or not.
- [`docs/red-team.md`](https://github.com/openscript-ai/openscript/blob/master/docs/red-team.md): every bypass found and what became
  of it.
- [`docs/metrics.md`](https://github.com/openscript-ai/openscript/blob/master/docs/metrics.md): every measurement, with its date and
  command.
- [`docs/submission.md`](https://github.com/openscript-ai/openscript/blob/master/docs/submission.md): the AI-safety challenge
  write-up, with each claim marked implemented, tested or theoretical.

## Limitations

**What it is not**

- **Not an injection detector to rely on.** It has pattern detectors, and they
  catch 2 to 7% of attacks they were not written for. The boundary is the
  rules and the approvals, and the rules are only as tight as you write them.
- **Not a compliance certification.** Its compliance presets assist a
  program and certify nothing.
- **Not a guard on what the agent says.** A reply that talks a person into
  acting is out of scope.

**Known weaknesses**

- **The effect side is not shipped.** It is a reference implementation in
  the test harness, not part of the package. Without it, the wrapper runs
  in the agent's process, and an agent that can run code can reach the
  function underneath.
- **A shell can undo the hook.** A shell the agent controls can rewrite the
  Claude Code hook's own files. That needs a boundary the agent does not
  own: Claude Code's sandbox, another OS user, or the effect side.
- **The session rule leaks.** It holds only after a read from a source you
  listed. It missed the attacks that arrive through Slack's channel listing:
  6 of 249 in the run above, and 27 of 617 in the no-model replay.
- **It costs attention.** Asking on every irreversible call stops 59 of 97
  ordinary tasks to ask a person.
- **The person is trusted.** An approver who says yes to everything lets
  every attack through, and how often people do that was not measured.

**Areas for improvement**

- **More models.** Containment was tested with one open 7B model, on two
  suites, with one attack wording and one sample per case.
- **Ship the effect side** in the package, with its key held outside the
  agent's process.
- **Ask less often,** since the cost of the strong claim is the main
  obstacle to using it.
- **A production deployment.** Countersign ran against a sandbox. Nothing
  here has carried real money or real mail.

## Quick start

```bash
pip install openscript-sdk          # Python 3.11+; pulls pydantic, structlog and PyYAML
```

The distribution is `openscript-sdk` and the package imports as
`openscript_sdk`. The extras are `[server]` (the FastAPI server, whose
database migrations are in this repository's `alembic/`), `[redis]`,
`[metrics]` and `[otel]`. To work on it, clone the repository and run
`make install`.

A tool that the rules hold
waits for a person, and the retry with the approval runs that exact call
once:

```python
from openscript_sdk import ActionBlockedError, SecureTool, ToolFirewallPolicy

async def transfer_funds(to: str, amount: int) -> str: ...

transfer = SecureTool(transfer_funds, policies=[ToolFirewallPolicy(rules_path="tools.yaml")])

call = {"to": "acct-22", "amount": 500}
try:
    await transfer.call(call)                       # held: the tool does not run
except ActionBlockedError as held:
    approval_id = held.approval_id                  # a person approves it ...

await transfer.call(call, approval_id=approval_id)  # ... and the same call runs, once
```

```yaml
# tools.yaml
default_deny: true
rules:
  transfer_funds:
    requires_approval: true
    args:
      amount: {min: 0, max: 1000}
```

Also in the package: redaction of secrets and PII, input screening, an audit
trail and an event server. The [guide](https://github.com/openscript-ai/openscript/blob/master/docs/guide.md) covers each.

## Where it runs

| Surface | Start here |
|---|---|
| Python agents and tools (LangChain, LangGraph, anything with `invoke`) | [the guide](https://github.com/openscript-ai/openscript/blob/master/docs/guide.md) |
| Claude Code, as a hook, with a strict profile that holds Bash after a web read | [the hook](https://github.com/openscript-ai/openscript/blob/master/docs/guide.md#the-claude-code-hook-what-it-adds-and-what-it-does-not-do) |
| Java agents, through the server | [`clients/java`](https://github.com/openscript-ai/openscript/blob/master/clients/java/README.md) |
| A browser, and a Chrome extension | [the demo](https://openscript-demo.vercel.app/) |

## Further reading

[`docs/guide.md`](https://github.com/openscript-ai/openscript/blob/master/docs/guide.md) covers every feature, its limits and its
configuration, including environment variables and options.

## Contributing

```bash
make install && pytest && ruff check . && black --check . && mypy --strict openscript_sdk/ --exclude openscript_sdk/server/
```

[CONTRIBUTING.md](https://github.com/openscript-ai/openscript/blob/master/CONTRIBUTING.md) has the full checklist, including the
Pyodide and Java checks. Changes to approvals, the session rule or the hook
also run `pytest tests/bypass`.

## License

Apache License 2.0. See [LICENSE](https://github.com/openscript-ai/openscript/blob/master/LICENSE).
