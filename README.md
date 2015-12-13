[![Build Status](https://travis-ci.org/edgurgel/verk.svg?branch=master)](https://travis-ci.org/edgurgel/verk)

Verk
===

Verk is a job processing system backed by Redis. It uses the same job definition of Sidekiq/Resque.

The goal is to be able to isolate the execution of a queue of jobs as much as possible.

Every queue has its own supervision tree:

* A pool of workers;
* A `QueueManager` that interacts with Redis to get jobs and enqueue them back to be retried if necessary;
* A `WorkersManager` that will interact with the `QueueManager` and the pool to execute jobs.

Verk will hold 1 connection to Redis per queue plus 1 dedicated to the `ScheduleManager`.

The `ScheduleManager` fetches jobs from the `retry` set to be enqueued back to the original queue when it's ready to be retried.

The image below is an overview of Verk's supervision tree running with a queue named `default` having 5 workers.

![Supervision Tree](http://i.imgur.com/4FDOOJH.png)

Feature set:

* Retry mechanism
* Dynamic addition/removal of queues
* Reliable job processing (RPOPLPUSH and Lua scripts to the rescue)

TODO:

* Error reporting (GenEvent?)
* Metrics (GenEvent?)
* Scheduled jobs
* Store dead jobs (too many retries)
* JSON API (external library?)

## Installation

First, add Verk to your `mix.exs` dependencies:

```elixir
def deps do
  [{:verk, "~> 0.1.0"}]
end
```

and run `$ mix deps.get`. Now, list the `:verk` application as your
application dependency:

```elixir
def application do
  [applications: [:verk]]
end
```

Verk was tested using Redis 2.8+

## Workers

A job is defined by a module and arguments:

```elixir
defmodule ExampleWorker do
  def perform(arg1, arg2) do
    arg1 + arg2
  end
end
```

This job can be enqueued using `Verk.enqueue/1`:

```elixir
Verk.enqueue(%Verk.Job{queue: :default, class: "ExampleWorker", args: [1,2]})
 ```

## Configuration

Example configuration for verk having 2 queues: `default` and `priority`

The queue `default` will have a maximum of 25 jobs being processed at a time and `priority` just 10.

```elixir
config :verk, queues: [default: 25, priority: 10],
              poll_interval: 5000,
              node_id: "1",
              redis_url: "redis://127.0.0.1:6379"
```

The configuration for releases is still a work in progress.

## Queues

It's possible to dynamically add and remove queues from Verk.

```elixir
Verk.add_queue(:new, 10) # Adds a queue named `new` with 10 workers
```

```elixir
Verk.remove_queue(:new) # Terminate and delete the queue named `new`
```

## Reliability

Verk's goal is to never have a job that exists only in memory. It uses Redis as the single source of truth to retry and track jobs that were being processed if some crash happened.

Verk will re-enqueue jobs if the application crashed while jobs were running. It will also retry jobs that failed keeping track of the errors that happened.

The jobs that will run on top of Verk should be idempotent as they may run more than once.