+++
title = "Cancellation should be more common"
date = "2024-09-05"
in_search_index = true
+++

## Why do I care about cancellation?

A lot of systems end up relying on asynchronous IO for at least _some_ of their work. I mostly deal with web servers so that means talking to caches, databases, and other services. Most clients for these resources provide timeout configurations. However, multiple steps of the operation could be misbehaving at once, or multiple async operations within the request could be degraded. To protect throughput, you likely want to set an overall timeout on your request handlers that depends on the expected runtime for those endpoints.

It is common in my experience to want to abort and clean up other operations that are still pending if one fails. For programs using `async/await` style idioms, coroutines, actors, or any [cooperative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking) mechanism, you'll likely need to signal failure to other processes and expect them to handle it.

My current employer also makes use of AWS Lambda which enforces an overall runtime duration on the serverless function. We're using Node.js for the Lambda functions. When your time runs out, execution is simply stopped. Sometimes that Lambda instance is reused to serve other requests and you get oddities like database connections immediately timing out and [finally blocks from previous runs](https://stackoverflow.com/questions/73285786/aws-lambda-node-js-finally-block-behaviour) executing. Metrics and traces can also get skewed in strange ways or lost entirely when this happens. Node has [AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) which functions as a cancellation token, but it's rare to see it accepted in the APIs of concurrent code outside of `fetch`. The Lambda context provides a function to get the remaining runtime, but to make use of that you need to pass it down to every async function you call and make a decision based on the remaining time.

Languages that provide concurrency primitives yet do not have an ecosystem that supports cooperative cancellation are shifting a lot of complexity onto you as a developer.

## Some examples

The following sections are how a few different ecosystems handle cancellation. In all of them I'm just going to execute a single SQL query that sleeps for a specified amount of time. This is obviously a little contrived, you could just set a timeout on your SQL driver to protect against this trivial case.

I'm doing this for simplicity. A single request is probably going to interact with several async tasks throughout its lifecycle and you should strive to limit the overall duration.

## Cancellation in Ruby on Rails

Ruby has a global interpreter lock so while you may have multiple threads running to handle requests, only one of them can be live at any given time. This isn't necessarily a problem since a lot of what happens in an application is IO activity which means the other threads can wake up and handle things while that operation is pending.

Most web servers for Ruby on Rails, like Puma, let you configure a count of threads and workers. Workers will fork the process and give you full parallelism on a single node. Even still, it is critical that requests complete in a timely manner to keep the application responsive. I've had a single slow endpoint take down an entire application with around 60 pods, 3 workers per pod. It was only accessible to a small subset of users, but they were getting frustrated and reloading the page which added to the pile of requests rapidly. We quickly shipped a "fix" to break the endpoint to salvage the rest of the application while we worked on fixing the performance.

There isn't an idiom of using cancellation tokens in Ruby, therefore ensuring your overall request duration stays within certain boundaries is tricky. _Most_ libraries that perform IO give you configurable timeouts you can set. This helps prevent any one step of the request from exceeding a timeout but you don't have many options for end to end timeouts.

Enter the [Timeout](https://ruby-doc.org/stdlib-3.0.1/libdoc/timeout/rdoc/Timeout.html) module from the standard library. You might use it like this:

```rb
def index
  Timeout.timeout(2) do
    seconds = params.require(:seconds).to_i
    ActiveRecord::Base.connection.exec_query("SELECT pg_sleep($1)", "SQL", [seconds])
    render plain: "Hello, world!"
  end
end
```

Let's start running through some tests with a well behaved client.

```
fetch('http://localhost:3000/timeouts?seconds=3');

Completed 500 Internal Server Error in 2002ms (ActiveRecord: 2001.1ms | Allocations: 1355)
Timeout::Error (execution expired):
```

This appears to give us the desired behavior. Whatever happens inside the `Timeout` block won't exceed the boundary we've set.

How about when the well behaved client gives up:

```
fetch('http://localhost:3000/timeouts?seconds=3', { signal: AbortSignal.timeout(500) })

Completed 500 Internal Server Error in 2002ms (ActiveRecord: 2000.6ms | Allocations: 1355)
Timeout::Error (execution expired)
```

There isn't any mechanism to detect this so the request runs the full duration regardless of whether the client has told us to give up or not. We'll just run to the server's timeout and give up. This means the resources consumed by that abandoned request can't be released for other requests. At least that `Timeout` block is preventing it from running forever and protecting the server. Or is it?

### Ruby Timeout is dangerous

The internet doesn't need another "Ruby's timeout is dangerous" blog post so I'll just give you a quick summary and point to better writers in the past. Ruby's Timeout is implemented by starting another thread to run a timer and then raising an exception in the original thread if that timer expires. This can end up terminating that block _anywhere_ in a very unsafe way.

You can find fallout of this in GitHub issues all over the internet. Here's one for [rack-timeout](https://github.com/zombocom/rack-timeout/issues/39). Here's [another one](https://github.com/zombocom/rack-timeout/issues/49).

The fallout spreads to Rails itself where it had [surprising effects in transactions](https://github.com/rails/rails/pull/29333). Trying to fix this lead to a breaking change in Rails 7 where [early returns in transactions now lead to rollbacks](https://github.com/rails/rails/issues/45017). I had assumed most Ruby devs already knew the risks of `Timeout` but the ongoing issues surrounding it tells me this isn't the case. Cancellation is a common need to protect your server and `Timeout` probably looks like an obvious choice.

More details on `Timeout`:

* [Julia Evans](https://jvns.ca/blog/2015/11/27/why-rubys-timeout-is-dangerous-and-thread-dot-raise-is-terrifying/)
* [Mike Perham](https://www.mikeperham.com/2015/05/08/timeout-rubys-most-dangerous-api/)
* [Changes to Timeout itself](https://github.com/ruby/timeout/pull/30). I found this especially interesting since it illustrates how language features like `next` that I only think of as iteration primitives are valid in _any_ block context and cause difficulties for libraries.

In short, you probably shouldn't be using `Timeout`.

### Safer timeouts?

I mentioned [Rack Timeout](https://github.com/zombocom/rack-timeout) above and it's important again in this section. They have a configuration option [term_on_timeout](https://github.com/zombocom/rack-timeout/blob/main/doc/settings.md#term-on-timeout) which will send a `SIGTERM` the the process serving requests. Web servers like Puma know to then restart that worker automatically. This avoids the sort of socket corruption, connection pool loss, and other issues described in the posts and GitHub issues above. However it also means when a request times out on your server you lose a worker and all the threads it had for handling requests until a new one can be booted. On Kubernetes this often also means that pods stop responding to health checks in a timely manner and you start losing pods which can quickly cascade into down time.

Another disadvantage to the `SIGTERM` option is that not all monitoring tools seem to handle it well. You may end up losing traces and metrics from the requests that you most need to inspect.

## Cancellation in Node.js

This post is going to be long, and I want to show some positive examples so I'll keep this section short. `AbortController` and `AbortSignal` exist in Node but there isn't currently a lot of ecosystem support for passing them around and having async tasks clean up in reponse to them. I hope this improves.

Node's common HTTP servers support specifying timeouts that keep the server _responsive_ but because there's no way to signal cleanup you end up leaking asynchronous tasks. It will be a race to see if you run out of memory or database connections first. In the case of a runtime like Lambda you end up with bizarre errors on the next invocation. In both cases you're probably going to have some corrupted state in your system at some point as a result of this.

Here's a [good post](https://betterstack.com/community/guides/scaling-nodejs/nodejs-timeouts/) that goes into more detail on how to set up timeouts and the limitations.

## Trying it in Elixir/Phoenix

Erlang's actor model of concurrency was built with long lived asynchronous systems in mind from the start. Tasks are spawned under a [supervision tree](https://www.erlang.org/doc/system/sup_princ.html) attached to a [process](https://www.erlang.org/docs/23/reference_manual/processes). Much like OS processes, they can respond to signals from the rest of the system. [Phoenix](https://www.phoenixframework.org/) is a popular web framework built on top of Elixir which is implemented in Erlang. Let's take a look at how this might handle cancellation.

```elixir
defmodule TimeoutsWeb.TimeoutController do
  use TimeoutsWeb, :controller
  alias Ecto.Adapters.SQL

  def index(conn, %{"seconds" => seconds}) do
    seconds = String.to_integer(seconds)
    query = "SELECT pg_sleep($1)"
    params = [seconds]
    t = Task.async(fn ->
      SQL.query(Timeouts.Repo, query, params)
    end)
    case Task.yield(t, 2_000) do
      {:ok, {:ok, _result}} ->
        send_resp(conn, 200, "Slept for #{seconds} seconds")
      {:ok, {:error, _reason}} ->
        send_resp(conn, 500, "Failed to execute sleep query")
      nil ->
        send_resp(conn, 503, "Failed to complete in time")
    end
  end
end
```

This spawns a [Task](https://hexdocs.pm/elixir/1.12/Task.html), which is just an Erlang process, from the current process. Similar to Node's `Promise.race` we're able to inspect if the task succeeded, failed, or timed out. In Erlang's case however, the task automatically receives an [exit signal](https://www.erlang.org/doc/system/ref_man_processes.html#sending-signals) from the Erlang runtime and has the opportunity to stop and clean up what it was doing. Let's test this out.

Calling it with a 1 second sleep:
```
[info] GET /timeout
[debug] Processing with TimeoutsWeb.TimeoutController.index/2
  Parameters: %{"seconds" => "1"}
  Pipelines: []
[debug] QUERY OK db=1002.0ms queue=0.2ms idle=1693.2ms
SELECT pg_sleep($1) [1]
[info] Sent 200 in 1002ms
```

Calling it with a 3 second sleep:
```
[info] Sent 503 in 2001ms
[info] Postgrex.Protocol (#PID<0.333.0>) disconnected: ** (DBConnection.ConnectionError) client #PID<0.795.0> exited
```

The SQL query that was running was immediately disconnected, cleaning up any ongoing work. A nice feature of the Elixir example is that it doesn't require any of the intermediate functions involved to accept a cancellation token. So long as the low level functionality like the database connection pool or http clients that have been invoked properly handle `exit`, resources will be freed when the process spawned by `Task.async` is aborted by `yield`.

If I make a request with a well behaved client that aborts the request however, such as with `fetch('http://localhost:4000/timeout?seconds=4', { signal: AbortSignal.timeout(500)})` the server still processes the request up until its own timeout at 2 seconds and attempts to return a 503. This is unfortunate, it would be nice to be able to release those resources sooner.

## Cancellation in C#/.NET

This is a controller method from a .NET app written with C# 8. In .NET land it's common for async operations to provide a function overload that accepts a `CancellationToken`. Controller methods provide one that is already bound to the request lifecycle and you can link from that to add a timeout.

```c#
[HttpGet]
public async Task<string> Timeout(CancellationToken abort, int seconds)
{
    var tokenSource = CancellationTokenSource.CreateLinkedTokenSource(abort);
    var duration = new SqlParameter("duration", $"00:00:{seconds:00}");
    tokenSource.CancelAfter(2000);  
    await _context.Users.FromSqlRaw($"WAITFOR DELAY @duration;select * from AspNetUsers", duration).ToListAsync(tokenSource.Token);
    return "Done";
}
```

Let's send a slow request: `fetch('https://localhost:7117/api/registration?seconds=3'`. In this case the error is `SqlException: A severe error occurred on the current command. The results, if any, should be discarded. Operation cancelled by user`. The query is immediately stopped and you are given the opportunity to handle it. Resources are released and any tasks handed a token from our source have the opportunity to clean up.

If you have a well behaved client that aborts, such as `fetch('https://localhost:7117/api/registration?seconds=2', { signal: AbortSignal.timeout(100)}) ` the controller action is immediately aborted with an exception `System.Threading.Tasks.TaskCanceledException: A task was canceled`. This also cleans up the query. This functionality would have saved our application in the scenario I mentioned above. If the user refreshes the page, the old request is aborted and the resources freed. Those users would have still been annoyed that part of the application wasn't responsive, but there weren't enough of them to cause a full outage.

## Trying it in Go

Here's an HTTP handler written in Go that grabs the seconds parameter from the URL, converts it to an integer, and hands it off to a query. I just used standard library code for this aside from installing a pg driver for SQL. You could skip the explicit timeout by using something like `TimeoutHandler` to wrap the function.

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	secondsStr := r.URL.Query().Get("seconds")
	seconds, err := strconv.Atoi(secondsStr)
	if err != nil {
		log.Printf("Error parsing seconds: %q", err)
		w.WriteHeader(http.StatusBadRequest)
		fmt.Fprint(w, "Invalid seconds")
		return
	}
	ctx, cancel := context.WithTimeout(r.Context(), time.Duration(2*time.Second))
	defer cancel()
	_, err = db.QueryContext(ctx, "SELECT pg_sleep($1)", seconds)
	if err != nil {
		log.Printf("Error querying database: %q", err)
		w.WriteHeader(http.StatusServiceUnavailable)
		fmt.Fprint(w, "Error querying database")
	} else {
		fmt.Fprintf(w, "Hello World")
	}
})
```

Let's run through the same sort of tests. A fetch with a duration under our timeout comes back just fine:

```
fetch('http://localhost:4000/?seconds=1')
2024/09/07 11:26:43 Query successful
```

A request that is too slow for our timeout causes the request to abort and the query immediately wraps up.

```
fetch('http://localhost:4000/?seconds=3')
2024/09/07 11:27:03 Error querying database: "pq: canceling statement due to user request"
```

If we send a slow request with a well behaved client that hangs up before we complete it will also immediately stop the query and free resources. I added a little extra logging on this one to clarify the behavior.

```
fetch('http://localhost:4000/?seconds=3', { signal: AbortSignal.timeout(1000)})
2024/09/07 11:36:24 Querying database
2024/09/07 11:36:25 Error querying database: "pq: canceling statement due to user request"
2024/09/07 11:36:25 Request complete
```

The context API in Go [isn't without controversy](https://faiface.github.io/post/context-should-go-away-go2/), but considering the alternatives that I'm often faced with I'm pretty happy to have it and see it widely used. Adding support for it [in existing libraries](https://github.com/vertica/vertica-sql-go/pull/81) often isn't very painful.

## Why do I care about well behaved clients?

When you're exposed to the internet it's basically guaranteed that many of the clients on your server are going to exhibit bad behavior. They might just abandon the network socket. They could also trickle headers or payload to you in a [Slowloris](https://www.cloudflare.com/en-gb/learning/ddos/ddos-attack-tools/slowloris/) attack. Go and C# both detect at least _some_ misbehaving clients and abort the request, but you'll want to make sure you configure your web server defensively.

I've worked on several applications that provide fairly expensive search functionality to analysts or power users. If they kick off something expensive and then suddenly change their mind it's nice to be able to abort that request in the client and have the server immediately stop and clean up the resources that were in use. In one case my team was allocated a _very_ small connection pool from a Vertica database so cleaning up unwanted searches was critical for keeping any sort of responsiveness.

## Wrapping up

Cancellation is an important part of building reliable systems. Adding timeouts to network requests and database operations only gets you part of the way there. Not having this functionality in the ecosystem you're developing in inevitably pushes a lot of complexity back up to you as a developer and may end up leaving you with no good options. If you're assessing technology for a new project, keep this in mind. If you're a library developer please ensure you're exposing cancellation mechanisms supported in your ecosystem.