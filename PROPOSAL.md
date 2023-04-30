This is an idea for improving the workflow of `iex -S mix phx.server`

## The Problem and The Solution in use

When running multiple Phoenix endpoints in an environment where you only have access to single port, we often:

- use a proxy like [main_proxy](https://github.com/Main-Proxy/main_proxy).
- set `server: false` for all the endpoints.

But, after `:server` option is set to `false`, the endpoint watchers won't start anymore - that's the problem.

I have noticed `:force_watchers` option and the [original PR](https://github.com/phoenixframework/phoenix/pull/4614).

Indeed, it works. But I think there is still room for improvement.

## A Potential Problem

Let's imagine a case first:

- I have an umbrella project contains multiple Phoenix endpoints.
- I configure main_proxy for these Phoenix endpoints.
- I set `server: false` for these endpoints.
- I set `force_watchers: true` for these endpoints.

Now, I want to enter the dev environment:

```
$ iex -S mix phx.server
```

Everything works fine.

But, sometimes, I want to enter the dev environment without starting watchers (for example, connect to an existing node which already starts watchers):

```
$ iex -S mix --sname bob
```

However, as it turns out, the watchers are still started. (After all, we used the `:force_watchers` option.)

In this case, `:force_watchers` can't help, and it brings conflicts on the running watchers. (For example, you are running two esbuild instances for the same assets, that's not what you want, generally.)

How to solve this problem?

## An Idea

I have a roughly formed idea:

- seperate the functionalities of endpoints into two:
  - web server
  - watchers
- `iex -S mix phx.server` controls web server and watchers at the same time:
  - That is because the underlying implementation is `Application.get_env(:phoenix, :serve_endpoints)`.
  - **It concerns endpoints**. And, as said above, endpoints have two functionalities - web server and watchers.
- `:server` option controls the web server
- `:force_watchers` option controls the watchers

A simple change on the code [here](https://github.com/phoenixframework/phoenix/blob/a310102eb0e20b918624bb2323a6afb124fcaddd/lib/phoenix/endpoint/supervisor.ex#L157):

```elixir
# unchanged
defp server?(conf) when is_list(conf) do
  Keyword.get_lazy(conf, :server, fn ->
    Application.get_env(:phoenix, :serve_endpoints, false)
  end)
end

# changed for this idea
defp watcher?(conf) do
  Keyword.get_lazy(conf, :force_watchers, fn ->
    Application.get_env(:phoenix, :serve_endpoints, false)
  end)
end

defp watcher_children(_mod, conf) do
  if watcher?(conf) do
    Enum.map(conf[:watchers], &{Phoenix.Endpoint.Watcher, &1})
  else
    []
  end
end
```

## Use Cases

Let's take another look and see if this idea can cover all general use cases.

### Enable web server and watchers

`iex -S mix phx.server`

Cases:

- normal Phoenix projects in dev environment
- ...

### Enable web server only

Set `server: true`.

Cases:

- normal Phoenix projects in prod environment
- ...

### Enable watchers only

- `iex -S mix phx.server`
- Set `server: false`

Cases:

- umbrella projects contains multiple proxied phoenix endpoints in dev environment
- ...

### Disable web server and watchers

- do nothing

Cases:

- umbrella projects contains multiple proxied phoenix endpoints in prod environment
- just want to run `iex -S mix`

## Last

Personally, I think this is a good proposal, which expands the applicability of `iex -S mix phx.server`. ;)

That's what I wanted to say. Thank you very much for reading. All feedback is welcome.

If you all like this proposal, I will create a pull request.
