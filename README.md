# Osysharp.LlmRouter — a cost-tiered model router

A policy for an app that declares more than one model: **capability filters, a quality floor per kind of work and per
step, a cost ceiling, then the cheapest model that clears them.** Name it as the app's router and every call to
Osysharp.Llm — `Llm.Complete`, `Llm.Stream`, `Llm.Extract<T>`, `Llm.Decide<T>` — goes through it, without a single call
site naming a model.

## Use case

An app whose work is not all the same size. Classifying an inbox, extracting fields from an invoice and drafting a reply
to a customer are three different jobs, and paying the top-tier price for all three is how a bill arrives that nobody can
explain. The usual fix — choosing a model at each call site — spreads a pricing judgement through the business logic and
makes "what do we spend this on, and why" a question you answer by reading code.

This kit makes it one declaration. You list the models the app has credentials for, what each can do, what it costs, and
a quality rank. Reasoning gets a floor; a vision task can only go to a model with vision; a nearly spent budget steps
down to a cheaper model; everything else takes the cheapest that clears. When somebody asks in March why one call went to
the expensive model, the decision carries the rule that made it and the reason each other candidate was dropped.

It has no user interface, so there is nothing to screenshot: it decides where calls go, and the calls look the same.

## Install and use

```osy
// app.osy
app Support {
  model "**/*.osy";
  use Osysharp.Llm@2;
  use Osysharp.LlmRouter@0;
  use Osysharp.Models.Anthropic@1 { egress "api.anthropic.com"; }
}
```

```osy
// config.osy
app.Llm = new LlmSetup {
  Models = [
    new LlmModel { Name = "Fast", Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-haiku-5-5",
                   Price = new LlmPrice { Input = 0.10m, Output = 0.50m } },
    new LlmModel { Name = "Strong", Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-opus-5-5",
                   Price = new LlmPrice { Input = 4, Output = 20 } },
  ],
  Router = new CostTieredRouter {
    Models = [
      new RoutedModel { Name = "Fast", Cost = 1, Quality = 1, Tools = true, StructuredOutput = true, Batch = true,
                        InputPrice = 0.10m, OutputPrice = 0.50m },
      new RoutedModel { Name = "Strong", Cost = 40, Quality = 3, Tools = true, Vision = true, StructuredOutput = true,
                        InputPrice = 4, OutputPrice = 20 },
    ],
    Floors = [ new QualityFloor { Kind = LlmTaskKind.Reason, Quality = 3 } ],
    StepFloors = [ new StepFloor { Step = LlmStep.Plan, Quality = 3 } ],
  },
};
```

`LlmSetup.Models` is what the app has credentials for; the router decides which of them serves each call, and in what
order the others are tried.

## What it decides, in order

The order is the design, and none of the steps commutes with another:

| | |
|---|---|
| 1. **Capability and window** | A model that cannot do the work is out, whatever it costs — `NeedsTools`, `NeedsVision`, `NeedsStructuredOutput`, a typed decision, and a prompt larger than `MaxInputTokens`. |
| 2. **The quality floor** | The highest of the floor for this kind of work (`Floors`), the floor for this step (`StepFloors`) and the `MinQuality` the call itself asks for. A floor is a refusal to be cheap about something, so it applies *before* price. |
| 3. **The cost ceiling** | When the call has one (`MaxCostMicros` on its facts), a model whose worst case costs more is out: the whole prompt at `InputPrice` plus the longest answer — the call's own `MaxOutputTokens`, else the model's — at `OutputPrice`. This is what lets a short budget **step down** to a cheaper model instead of stopping. A model with no price declared is never excluded by it. |
| 4. **The call site's request, narrowing only** | If the model it asked for survived 1 to 3, that serves alone. If it asked for something the policy does not admit **for this task**, the call is DENIED — not quietly given something else. |
| 5. **The order** | A decision model first for a decision (`Decides`); a better model than the last one first when the call is asking again; a batch endpoint first when nobody is waiting (`LatencySensitivity = Batch`); then cheapest, the declared order breaking a tie. |

The order it answers is also the **fallback order**: when the first model's provider fails, the call moves to the next.
`Fallbacks` caps how many may follow the first (`null`: every admissible one; `0`: none). Every model dropped along the
way is recorded with the reason it was dropped, and a task nothing can serve is a denial carrying those reasons — never a
substitution.

The rule id on each decision: `cost-tiered:2` for an ordinary one, `cost-tiered:requested` when the call's own choice
served, `cost-tiered:narrowing` when it was denied, `cost-tiered:escalated` when the call was asking again.

## The fields

`RoutedModel` — one model the policy may serve with:

| field | meaning |
|---|---|
| `Name` | exactly as `LlmSetup.Models` names it — this is what the call is served with |
| `Cost` | relative cost, in any unit, as long as it is the same unit for every entry. The router only ever COMPARES them. |
| `Quality` | a rank this app assigns, higher being better. Used only against a floor — not a score to read as absolute. |
| `Tools` · `Vision` · `StructuredOutput` | what it can do. Each `false` is a hard exclusion for a call that needs it. |
| `Decides` | it is built for typed decisions (`LlmModel.Decider`). A decision goes to such a model first; a chat model that produces structured output may stand in behind it. |
| `MaxInputTokens` | the largest prompt it accepts. `0` means the app declares no limit — deliberately not a guess, because guessing a window low denies work that would have run. |
| `Batch` | whether it has a cheap batch endpoint. Preferred for a `Batch` call; nothing is ever denied for lacking one. |
| `InputPrice` · `OutputPrice` | dollars per million tokens, as on the model's `LlmPrice`. What the cost ceiling is checked against. Both `0`: unknown, never excluded. |
| `MaxOutputTokens` | the longest answer assumed for the ceiling when the call does not say (default `4096`). |

`QualityFloor` is the lowest `Quality` that may serve one `LlmTaskKind`; `StepFloor` the lowest for one `LlmStep` of a
larger job — "plan with rank 3, fill with anything". A kind or step with no floor accepts anything that clears the
filters.

## Why `Route` takes no I/O

`Route` is a pure function of the facts and the router's own fields — no data read, no data write, no egress.
Two things depend on it:

- **Shadow routing** — replay stored facts against a candidate policy and diff the decisions before adopting it.
- **Reproduction** — the same facts and the same policy reach the same decision, forever.

Both are properties of a function. That is why everything this router knows is on the fields you set where it is named,
and why changing a policy is a recompile rather than a row edit.

## Tested

`kits/llm-router/tests` is a print shop with three models and a decision model, and a policy over them. Thirteen tests
ask the policy directly, because it is a pure function of the facts — the cheapest first with the rest as fallbacks,
capability as a filter whatever the price, the three floors with the highest winning, the cost ceiling, narrowing, a
decision model with the chat models behind it, asking again, the batch endpoint, the `Fallbacks` cap, a router with no
models, and the same facts reaching the same decision — and two go through a real call with scripted models: a provider
that fails handing the call to the next in the policy's order, and a short budget stepping down to the model the
ceiling admits.

```
osy test kits/llm-router/tests
```

## Source and package

`model/CostTieredRouter.osy` — 243 lines of ordinary Osy#, and the whole implementation.

```
package Osysharp.LlmRouter {
  version "0.2.0";
  use Osysharp.Llm@2;
}
```

No egress, no secret, no data. A routing policy needs none of them, which is exactly why it can be a package and still
be the thing that decides where all of an app's model calls go.

## See also

[`Osysharp.Llm`](../llm) — the calls, the meter, budgets and what a router is handed.
