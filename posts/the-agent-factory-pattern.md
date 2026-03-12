---
title: 'The Agent Factory: Building Consistent Algolia Agents at Scale'
description: How to stamp out correctly-configured agents across multiple contexts using the Agent Factory pattern — template-driven configuration, registry-backed IDs, and full lifecycle management.
tags: 'agents, patterns, devops'
cover_image: ./assets/agent_factory_header_1000x420.png
canonical_url: null
published: false
---

Agents are suddenly everywhere. But deploying the same agent to multiple events, customers, or product lines is harder than it looks. How do you make sure every instance has the right configuration? And when something improves, how do you propagate it without touching each one by hand?

I developed the Agent Factory pattern scaling a conference demo from one event to many. This post covers the pattern — and how to apply it to any project where agents need to be deployed consistently across multiple contexts.

---

## The Three-Legged Stool

Before describing the pattern, it helps to be precise about what an agent actually is. Strip away the API surface and every agent is built from three things:

**Model** — the LLM provider and model name. This is the reasoning engine. It determines response quality, context window, and cost per query.

**Tools** — what the agent can do. For an Algolia-backed agent, this is typically an `algolia_search_index` tool pointing at one or more indices. The index descriptions aren't just metadata — the model reads them to understand what each index contains and when to query it.

**Prompt** — the system instructions that shape how the agent behaves: its role, its tone, its constraints, and critically, how it should use its tools. This is where domain knowledge lives.

Typically in agent frameworks all three legs are configured at creation time. That's what makes agents predictable and auditable — each one is a known, stable configuration rather than a dynamic runtime object. But it also means that when you need the same agent experience across multiple contexts, each context needs its own correctly-configured instance. At meaningful scale, managing that by hand doesn't hold up.

---

## When One Stool Isn't Enough

Recently, I used [Algolia's Agent Studio](https://www.algolia.com/doc/guides/algolia-ai/agent-studio) to build a demo application for a Pokemon card vending machine we use at Algolia's conference booths. Attendees chat with an AI agent to find cards, check values, and claim what they received. The agent retrieves all of the underlying data from a searchable index using an MCP-like tool.

Each conference gets a fresh machine stocked with that event's cards. All three legs of the stool need to adapt: the **tool** must point at that event's index (`shoptalk-2026`, not `etail-palm-springs-2026`). The **prompt** must reference the right event name, booth number, and context. Even the underlying **model** might change based on cost or event context (I should probably steer clear of Gemini models at an AWS event!).

All three parts of the definition can vary per event. So how do you stamp out a correctly-configured agent for each event without doing it by hand?

At first, I considered a single shared agent with tool access to every event's indices. It's tempting to have one agent to maintain. But agents are static. A shared agent has no way to know which event it's at, so it can't scope its searches or tailor its responses to the right context. It either searches everything indiscriminately or requires guardrails you can't actually enforce. One agent per event was the only way to provide this context. But how to keep them consistent?

---

## The Factory Pattern

### One Template, Three Legs

The key insight is that the model, tools, and prompt aren't independent — they reference the same context. The index name in the tool configuration should match what the prompt tells the agent it's searching. If you render them separately, they can drift.

Since Agent Studio uses APIs for agent creation, the solution was to store everything as templates with shared placeholder variables and render all three legs together in a single pass at creation time:

**`agent-config.json`**
```json
{
  "name": "Demo Agent — {{event_name}}",
  "provider": "your-llm-provider",
  "model": "your-model",
  "instructions": "PROMPT.md",
  "tools": [
    {
      "type": "algolia_index_search",
      "index": "catalog_{{event_id}}",
      "description": "Product catalog for the {{event_name}} event."
    }
  ]
}
```

**`PROMPT.md`** (excerpt)
```markdown
You are an agent at {{event_name}} (booth {{booth}}).
Use Algolia search to help attendees find cards in the vending machine.
```

`{{event_id}}`, `{{event_name}}`, and `{{booth}}` are resolved across both files together. A missing or mismatched variable fails before any API call is made. When the prompt improves or the tool configuration changes, both are updated atomically.

### Registering What's Been Built

Agent Studio hosts your agents as managed endpoints, each with a unique ID returned at creation time. That ID is how your application loads the right agent at runtime, but it isn't predictable in advance — you need to store it somewhere your application can find it. The factory writes each returned `agent_id` back to an event registry, closing the loop between creation and runtime. The registry could be a database, a config file, or any other store your application already reads from — in our case, another Algolia index.

### Operationalizing the Factory

Agent creation is only the beginning. When the prompt changes or new models drop, existing agents need to be updated. When an event is retired, its agent should be deleted. The factory exposes the full lifecycle:

```bash
# New event
python agent.py create shoptalk-2026 "Shoptalk 2026" 701 --publish

# Prompt improvement — preview then push to all existing agents
python agent.py update --all --dry-run
python agent.py update --all --publish

# Retire an event
python agent.py delete etail-palm-springs-2026
```

The `update --all` is the beating heart of the factory. One command re-renders the current templates against every event that has an agent. A prompt improvement or tool fix is now one command rather than N error-prone dashboard edits.

### Tracking What's Changed

Storing my prompt and configuration as files gave me the full benefits of version control: a history of what changed and why, the ability to roll back a prompt that regressed, and a clear record of what was running at any given time. I could compare behavior before and after a model upgrade, review a prompt change the same way I'd review a code change, and reason about what actually improved things.

This is especially useful now as the agent space is moving fast — model capabilities improve, better prompting patterns emerge, the tools themselves evolve. That discipline is easy to skip when you're managing agents through a dashboard. 

---

## Codifying the Factory

As the factory matured, a clear split emerged in the code:

**Generic** — HTTP calls to Agent Studio, provider resolution, rendering templates, diffing for updates, publishing, deleting. None of this knows anything about Pokemon cards or conference events.

**Domain-specific** — verifying the event exists in the event registry, constructing index names from the event ID, writing the `agent_id` back to the registry. All of this is unique to the demo.

That split is the pattern stated as code. I extracted the generic layer into a standalone open-source CLI — [`algolia-agent`](https://github.com/algolia-samples/algolia-agent-cli) — and a thin demo wrapper that calls the CLI's API and only contains the three things unique to its domain.

The CLI makes the factory pattern accessible for any project:

```bash
algolia-agent init        # defines all three parts of your agent interactively
algolia-agent create      # renders templates and calls the API
algolia-agent update      # diffs and pushes changes to existing agents
algolia-agent publish     # promotes a draft to live
```

The `init` command walks through the definition in order: pick a provider and model, select tools, write a prompt. It also offers a `create without tools` option since an agent backed only by a model and a prompt is a valid configuration.

### Building a Better Stool: A prompt example

A user at the booth asked: *"Do you have any cards worth more than $50?"*

The first version of the prompt had no guidance on numeric filters. The agent searched for "cards worth more than 50 dollars" as a keyword query. It got results, but not the right ones — keyword search matched card descriptions that mentioned the word "worth," not cards filtered by actual value.

The fix was one line in the prompt:

```markdown
For "more than" / "less than" questions on numeric fields, use comparison operators
in searchParams filters rather than facets: e.g. `searchParams: { filters: 'estimated_value > 50' }`
```

After adding this, the agent correctly translated the question into a filter against the `estimated_value` attribute, returning only cards actually priced above $50.

Since I was running agents for multiple active events, one `update --all` pushed the improvement to every instance simultaneously. The agent at the next conference was better than the one at the previous conference without anyone touching it individually. That's the payoff of treating the prompt as a versioned artifact and the fleet as a unit.

---

## Quality Control

The hardest part of managing multiple agents isn't creating them — it's keeping them consistent after the fact. Before pushing any change to the fleet, you want to preview exactly what will change: how the prompt shifted, which tools were added or removed, whether the model changed. A wrong change pushed to N agents is N times harder to debug than one caught before deployment. In my CLI, I included a `--dry-run` flag to show a diff of what changed before calling the API.

---

## Conclusion

Building an agent demo that needed to scale across conference events forced me to think carefully about what an agent actually is. Strip it down and you get three legs: a model, a set of tools, and a prompt. Once I had that definition, it was easy to separate what was core agent configuration from what was event-specific context.

The factory pattern helped me formalize that separation into something repeatable: scaffold the agent definition, render it for each context, perform consistent CRUD operations across every instance. When something improves — a better prompt, a model upgrade, a tool fix — one command propagates it to every agent at once.

Feel free to try out my [`algolia-agent`](https://github.com/algolia-samples/algolia-agent-cli) for yourself. I'd also love to hear if the factory pattern is useful to you, or how it can be improved!
