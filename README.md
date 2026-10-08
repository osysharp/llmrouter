# Osysharp.LlmRouter — a cost-tiered model router

A policy for an app that declares more than one model: **capability filters, a quality floor per kind of work, then the
cheapest model that clears them.** Register it and every model call in the app goes through it — `Llm.Extract`, an agent
turn, a `stream<T>`, all of them — without a single call site naming a model.

## Use case

An app whose work is not all the same size. Classifying an inbox, extracting fields from an invoice and drafting a reply
to a customer are three different jobs, and paying the top-tier price for all three is how a bill arrives that nobody can
explain. The usual fix — choosing a model at each call site — spreads a pricing judgement through the business logic and
makes "what do we spend this on, and why" a question you answer by reading code.

This kit makes it one declaration. You list the models the app has credentials for, what each can do, what it costs
relative to the others, and a quality rank. Reasoning gets a floor; a vision task can only go to a model with vision;
everything else takes the cheapest that clears. When somebody asks in March why one call went to the expensive model, the
decision carries the rule that made it and the reason each other candidate was dropped.

It has no user interface, so there is nothing to screenshot: it decides where calls go, and the calls look the same.

## Install and use

```osy
// app.osy
app Support {
  model "**/*.osy";
  use Osysharp.LlmRouter;
}
```

```osy
// config.osy
app.Models = [
  new LlmConfig("Cheap")  { Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-haiku-4-5" },
  new LlmConfig("Strong") { Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-sonnet-5" },
];

app.LlmRouter = new CostTieredRouter {
  Models = [
    new RoutedModel { Name = "claude-haiku-4-5", Cost = 1,  Quality = 1, Tools = true, StructuredOutput = true, Batch = true },
    new RoutedModel { Name = "claude-sonnet-5",  Cost = 15, Quality = 3, Tools = true, Vision = true, StructuredOutput = true },
  ],
  Floors = [ new QualityFloor { Kind = LlmTaskKind.Reason, Quality = 3 } ],
};
```

That is the whole configuration. `app.Models` is what the app has credentials for; the router decides which of them
serves each task.

## What it decides, in order

The order is the design, and none of the four steps commutes with another:

| | |
|---|---|
| 1. **Capability and window** | A model that cannot do the work is out, whatever it costs — `NeedsTools`, `NeedsVision`, `NeedsStructuredOutput`, and a prompt larger than `MaxInputTokens`. |
| 2. **The quality floor for this kind of work** | A floor is a refusal to be cheap about something, so it applies *before* price rather than being weighed against it. |
| 3. **The call site's request, narrowing only** | If what it asked for survived 1 and 2, that is what serves. If it asked for something the policy does not admit **for this task**, the call is DENIED — not quietly given something else. |
| 4. **Cheapest first** | And a batch endpoint preferred when nobody is waiting (`LatencySensitivity = Batch`), because that is what a batch tier is for. |

Every model dropped along the way is recorded with the reason it was dropped. A task nothing can serve is a denial
carrying those reasons — never a substitution.

## The fields

`RoutedModel` — one model the policy may serve with:

| field | meaning |
|---|---|
| `Name` | exactly as `app.Models` / `app.DefaultModel` declares it — this is what the platform serves with |
| `Cost` | relative cost, in any unit, as long as it is the same unit for every entry. The router only ever COMPARES them. |
| `Quality` | a rank this app assigns, higher being better. Used only against a declared floor — not a score to read as absolute. |
| `Tools` · `Vision` · `StructuredOutput` | what it can do. Each `false` is a hard exclusion for a task that needs it. |
| `MaxInputTokens` | the largest prompt it accepts. `0` means the app declares no limit — deliberately not a guess, because guessing a window low denies work that would have run. |
| `Batch` | whether it has a cheap batch endpoint. Preferred for a `Batch` task; nothing is ever denied for lacking one. |

`QualityFloor` — the lowest `Quality` that may serve one `LlmTaskKind`. A kind with no floor accepts anything that clears
the filters.

## Why `Route` takes no I/O

`Route` is pure — no data read, no data write, no egress — and the platform refuses a router that tries, naming what it
touched. Two things depend on it:

- **Shadow routing** — replay stored facts against a candidate policy and diff the decisions before adopting it.
- **Reproduction** — the same facts and the same policy version reach the same decision, forever.

Both are properties of a function. That is why everything this router knows is on the two fields you set at registration,
and why changing a policy is a recompile rather than a row edit.

## Source

`model/CostTieredRouter.osy` — 163 lines of ordinary Osy#, and the whole implementation. The platform's own suite
compiles it and routes through it, so what is documented here is what the file does.

## Package

```
package Osysharp.LlmRouter {
  version "0.1.0";
  use Osysharp.Llm;
}
```

No egress, no secret, no data. A routing policy needs none of them, which is exactly why it can be a package and still
be the thing that decides where all of an app's model calls go.

## See also

- `osy docs agent-model-routing` — the seam, the narrowing rule and deny-by-default
- `osy docs agent-models` — declaring the models an app admits
- `osy docs agent-llm-budget` — what a call may spend, measured above this seam
