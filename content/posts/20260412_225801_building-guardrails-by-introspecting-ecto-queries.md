+++
title = "Building Guardrails by Introspecting Ecto Queries"
slug = "building-guardrails-by-introspecting-ecto-queries"
description = "A practical approach to enforcing query safety and policy through Ecto query introspection."
date = 2026-04-12T22:58:01+03:00
draft = false

[taxonomies]
tags = ["engineering", "elixir"]

[extra]
toc = true
comment = false
+++

# Problem

Some query bugs are too semantic for source-level linting and too application-specific for database constraints.

In the age of agentic coding, I am looking more and more for ways to help AI produce code that is safe, readable, maintainable, secure, and performant.
That naturally pushes me toward guardrails at multiple levels:
- source-level checks
- query-level checks
- database-level checks
- and anything else that helps the system fail earlier and more clearly

Examples:
- a bulk [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) that forgets `updated_at`
- a composed join query that has become difficult to read or safely extend
- a timestamp range filter that uses the wrong boundary and silently drops rows

These are real guardrail problems:
- Credo can see the source code, but it cannot always reason about the final composed query
- PostgreSQL can enforce hard invariants, but many application guardrails need explicit escape hatches
- raw SQL inspection happens too late and loses the higher-level Ecto structure

# Solution

The core idea is to treat [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3) as a narrow runtime guardrail hook.

By the time a query reaches [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3), Ecto has already:
- expanded the query macros
- composed the final `%Ecto.Query{}`
- preserved a lot of useful semantic structure

That creates the middle layer we need:
- Credo can validate source shape
- PostgreSQL can validate database invariants
- `%Ecto.Query{}` can validate the final query shape right before execution

`%Ecto.Query{}` is a particularly good surface for guardrails because it is structured, introspectable, and preserved all the way to a single chokepoint in [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3).

The final query object still exposes the information we care about:
- `wheres`
- `joins`
- `order_bys`
- `limit`
- `updates`
- root schema metadata

Relevant official docs:
- [Ecto multi-tenancy with foreign keys](https://hexdocs.pm/ecto/multi-tenancy-with-foreign-keys.html)

Principles:
- only add checks whose signal survives into `%Ecto.Query{}`
- keep the initial rollout conservative
- prefer positive flags such as `ecto_query_runtime_checks: [validate_tenant_scope: false]`
- let checks return errors, and let `Repo` decide whether to raise, log, or emit telemetry
- keep an explicit escape hatch for the cases where the rule should not apply

Useful boundaries:
- source-level style belongs in Credo
- hard invariants with no legitimate exception usually belong in PostgreSQL
- final executed query-shape checks belong in [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3)

## Step 1: Inspect the final `%Ecto.Query{}`

The first step is to stop thinking in terms of source code shape and start looking at the final query object.

Take this query built with [`where/3`](https://hexdocs.pm/ecto/Ecto.Query.html#where/3), [`update/3`](https://hexdocs.pm/ecto/Ecto.Query.html#update/3), and [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3):

```elixir
query =
  MyApp.Conversation
  |> where([c], c.workspace_id == ^workspace_id)
  |> where([c], not is_nil(c.title))
  |> update([c], set: [title: "Archived"])

Repo.update_all(query, [])
```

By the time this reaches [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3), Ecto has already composed the final `%Ecto.Query{}`.

Here is the kind of shape you can expect when you inspect that query:

```elixir
#Ecto.Query<from c0 in MyApp.Conversation,
 where: c0.workspace_id == ^"...",
 where: not is_nil(c0.title), update: [set: [title: "Archived"]]>

%Ecto.Query{
  from: %Ecto.Query.FromExpr{
    source: {"conversations", MyApp.Conversation},
    as: nil,
    prefix: nil,
    params: [],
    hints: []
  },
  joins: [],
  wheres: [
    %Ecto.Query.BooleanExpr{...},
    %Ecto.Query.BooleanExpr{...}
  ],
  order_bys: [],
  limit: nil,
  updates: [
    %Ecto.Query.QueryExpr{
      expr: [
        set: [
          title: %Ecto.Query.Tagged{type: {0, :title}, value: "Archived"}
        ]
      ],
      params: []
    }
  ]
}
```

For this particular guardrail, the important parts are:
- `query.from` tells us which schema the bulk update targets
- `query.wheres` tells us which filters survived into the final query
- `query.updates` tells us which fields the bulk update is actually setting

This is the important shift:
- we are validating the final query object
- we are not trying to reconstruct how the source code was authored

## Step 2: Pick one guardrail with a direct mapping to a real bug

The cleanest first example is [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) and `updated_at`.

Why this one is good:
- it catches a real footgun
- it is easy to explain
- the signal survives cleanly into `%Ecto.Query{}`
- the fix is obvious

The underlying bug is simple:

```elixir
MyApp.Conversation
|> where([c], c.workspace_id == ^workspace_id)
|> Repo.update_all(set: [title: "Archived"])
```

This query updates rows, but it leaves `updated_at` stale.

## Step 3: Implement the check

This is a full concrete check adapted to the pattern above.

```elixir
defmodule MyApp.EctoQueryRuntimeChecks.UpdateAllSetsUpdatedAt do
  @behaviour MyApp.EctoQueryRuntimeChecks.Check

  @spec validate(
          operation :: atom(),
          query :: Ecto.Query.t(),
          opts :: Keyword.t()
        ) :: :ok | {:errors, nonempty_list(String.t())}
  def validate(operation, %Ecto.Query{} = query, opts \\ []) do
    cond do
      operation != :update_all ->
        :ok

      not Keyword.get(opts, :validate_update_all_updated_at, true) ->
        :ok

      not schema_defines_updated_at?(query) ->
        :ok

      updated_at_set?(query) ->
        :ok

      true ->
        {:errors,
         [
           "`Repo.update_all/3` must explicitly set `updated_at` when the target schema defines that field"
         ]}
    end
  end

  defp schema_defines_updated_at?(%Ecto.Query{} = query) do
    schema = schema_module(query)

    function_exported?(schema, :__schema__, 1) and
      :updated_at in schema.__schema__(:fields)
  end

  defp schema_module(%Ecto.Query{from: %{source: {_source, schema}}}) when is_atom(schema) do
    schema
  end

  defp schema_module(%Ecto.Query{}) do
    nil
  end

  defp updated_at_set?(%Ecto.Query{} = query) do
    Enum.any?(query.updates, &updated_at_set_in_expr?/1)
  end

  defp updated_at_set_in_expr?(%Ecto.Query.QueryExpr{} = query_expr) do
    query_expr.expr
    |> Keyword.get_values(:set)
    |> Enum.any?(&Keyword.has_key?(&1, :updated_at))
  end
end
```

What this check does, step by step:
- confirm the operation is `:update_all`
- allow an explicit opt-out
- inspect the target schema through [`__schema__/1`](https://hexdocs.pm/ecto/Ecto.Schema.html)
- check whether `updated_at` exists on that schema
- inspect `query.updates`
- fail if `updated_at` is missing from the final update expression

## Step 4: Wire it into [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3)

Once the check exists, [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3) becomes the single integration point:
- gather runtime-check opts
- skip Ecto internals such as preload-generated queries where appropriate
- run the checks
- raise in test/dev or emit advisory findings in production

That is what makes this approach practical.

You write the rule once and every matching query passes through the same chokepoint.

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres

  alias MyApp.EctoQueryRuntimeChecks

  @impl Ecto.Repo
  def prepare_query(operation, query, opts) do
    runtime_check_opts =
      Keyword.get(opts, :ecto_query_runtime_checks, [])

    repo_opts = Keyword.delete(opts, :ecto_query_runtime_checks)

    run_query_runtime_checks? =
      Keyword.get(runtime_check_opts, :run_query_runtime_checks, true)

    ecto_query = Keyword.get(repo_opts, :ecto_query)

    if query_runtime_checks_enabled?() and run_query_runtime_checks? and ecto_query != :preload do
      case EctoQueryRuntimeChecks.validate(operation, query, runtime_check_opts) do
        :ok -> :ok
        {:errors, errors} -> raise EctoQueryRuntimeChecks.Error, operation: operation, errors: errors
      end
    end

    {query, repo_opts}
  end

  defp query_runtime_checks_enabled? do
    :my_app
    |> Application.get_env(:query_runtime_checks, [])
    |> Keyword.get(:enabled, false)
  end
end
```

Example call site:

```elixir
Repo.all(
  query,
  ecto_query_runtime_checks: [
    validate_required_scope: false
  ]
)
```

## Rollout

Start in test:

```elixir
# config/test.exs
config :my_app, :query_runtime_checks,
  enabled: true
```

Later on, the same layer can also run in development and eventually in production in advisory mode.

## Check Behaviour

```elixir
defmodule MyApp.EctoQueryRuntimeChecks.Check do
  @callback validate(
              operation :: MyApp.EctoQueryRuntimeChecks.operation(),
              query :: Ecto.Query.t(),
              opts :: Keyword.t()
            ) ::
              :ok | {:errors, nonempty_list(String.t())}
end
```

# Examples

## Require named bindings on joins

Why it is useful:
- survives into `%Ecto.Query{}`
- low false-positive rate
- improves readability in larger composed queries

This example uses [`from/2`](https://hexdocs.pm/ecto/Ecto.Query.html#from/2), [`join/5`](https://hexdocs.pm/ecto/Ecto.Query.html#join/5), [`where/3`](https://hexdocs.pm/ecto/Ecto.Query.html#where/3), [`Repo.all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:all/2), and [`Ecto.assoc/2`](https://hexdocs.pm/ecto/Ecto.html#assoc/2).

```elixir
# wrong
from(c in MyApp.Conversation)
|> join(:inner, [c], m in assoc(c, :messages))
|> where([c, m], c.workspace_id == ^workspace_id and m.role == "assistant")
|> Repo.all()
```

```elixir
# better
from(c in MyApp.Conversation, as: :conversation)
|> join(:inner, [conversation: c], m in assoc(c, :messages), as: :message)
|> where([conversation: c], c.workspace_id == ^workspace_id)
|> where([message: m], m.role == "assistant")
|> Repo.all()
```

What the guardrail is checking:
- the root source has a name
- every join has a name
- the executed query remains easy to extend and reason about

This is a good runtime check because the final `%Ecto.Query{}` still preserves that information.

## Require [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) to set `updated_at`

Why it is useful:
- [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) bypasses Ecto timestamp autogeneration
- this is a real footgun
- it is easy to validate from the final query shape

```elixir
# wrong
MyApp.Conversation
|> where([c], c.workspace_id == ^workspace_id)
|> Repo.update_all(set: [title: "Archived"])
```

```elixir
# better
MyApp.Conversation
|> where([c], c.workspace_id == ^workspace_id)
|> Repo.update_all(set: [title: "Archived", updated_at: DateTime.utc_now(:microsecond)])
```

What the guardrail is checking:
- the target schema defines `updated_at`
- the executed [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) actually sets it

This is a particularly nice example because the query shape maps directly to a real bug.

## Require tenant or workspace scope

Why it is useful:
- high-value safety rule
- catches missing scope at the final executed query
- can be introduced in phases

Phase 1:
- only verify that some scope filter exists

Phase 2:
- compare the query scope against process context if needed

This example uses [`where/3`](https://hexdocs.pm/ecto/Ecto.Query.html#where/3) and [`Repo.all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:all/2).

```elixir
# wrong
MyApp.Conversation
|> where([c], not is_nil(c.title))
|> Repo.all()
```

```elixir
# better
MyApp.Conversation
|> where([c], c.workspace_id == ^workspace_id)
|> where([c], not is_nil(c.title))
|> Repo.all()
```

Or, in a tenant-oriented system:

```elixir
MyApp.Conversation
|> where([c], c.tenant_id == ^tenant_id)
|> where([c], not is_nil(c.title))
|> Repo.all()
```

What the guardrail is checking:
- schemas with scope fields such as `tenant_id` or `workspace_id` are actually filtered by them

This is where the [Ecto multi-tenancy with foreign keys](https://hexdocs.pm/ecto/multi-tenancy-with-foreign-keys.html) article is relevant.

## Use consistent timestamp range boundaries

Why it is useful:
- timestamp filtering is full of off-by-one bugs
- teams often benefit from one shared convention
- the executed query shape still exposes the comparison operators being used

One convention that works well is:
- use `>=` for the lower bound
- use `<` for the upper bound
- express time windows as `[start, end)`

This example uses [`where/3`](https://hexdocs.pm/ecto/Ecto.Query.html#where/3) and [`Repo.all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:all/2).

```elixir
# wrong
MyApp.Conversation
|> where([c], c.inserted_at >= ^month_start)
|> where([c], c.inserted_at <= ^month_end)
|> Repo.all()
```

```elixir
# better
MyApp.Conversation
|> where([c], c.inserted_at >= ^month_start)
|> where([c], c.inserted_at < ^next_month_start)
|> Repo.all()
```

Why the second version is better:
- it avoids end-of-day and microsecond edge cases
- it composes naturally across days, weeks, and months
- it gives the team one predictable way to express timestamp ranges

This is a good example of a guardrail that can encode a team convention around a real SQL pitfall.

# Tradeoffs and limitations

Important limitations to call out:
- it relies on `%Ecto.Query{}` internals that are powerful but not a long-term public stability contract
- this only covers code paths that go through [`Repo.prepare_query/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:prepare_query/3)
- it is strongest for [`Repo.all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:all/2), [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3), [`Repo.delete_all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:delete_all/2), [`Repo.stream/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:stream/2), and similar query-backed operations
- it is not a general solution for [`Repo.insert/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:insert/2), [`Repo.update/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update/2), or function-based [`Repo.transact/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:transact/2)
- some helpers may collapse into the same final query shape, which means source-level stylistic checks still belong in Credo
- Ecto internals such as preload queries may need special handling

Useful framing:
- this is a runtime **query-shape** guardrail layer, not a universal Elixir linting layer
- if PostgreSQL can enforce an invariant cleanly, that is often better
- if a rule needs frequent opt-outs, it is probably too noisy or should default to being disabled
- runtime checks are strongest when the signal survives into `%Ecto.Query{}`

That said, this tradeoff is often acceptable in practice:
- these checks live in a guardrail layer rather than in core business logic
- if Ecto changes an internal query shape in a future release, the failure mode is usually localized and easy to remove or adapt
- the worst case is typically that a guardrail stops working and needs to be updated, not that the application model itself becomes invalid

Other checks that follow the same pattern:
- reject [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3) / [`Repo.delete_all/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:delete_all/2) with no [`where/3`](https://hexdocs.pm/ecto/Ecto.Query.html#where/3)
- immutable field updates in [`Repo.update_all/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:update_all/3)
- deterministic ordering for high-risk reads
- SQL footguns around timestamps, ranges, floats, and money

# Closing thoughts

- Ecto query introspection is powerful because it gives one narrow chokepoint with real semantic information
- the best checks are conservative, explicit, and easy to opt out of
- the runtime layer is most useful when it works alongside Credo and PostgreSQL
- start with one or two high-signal checks, prove they are low-noise, then expand carefully
