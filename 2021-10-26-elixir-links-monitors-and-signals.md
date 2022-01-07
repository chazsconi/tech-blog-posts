---
layout: post
title:  "Elixir links, monitors and signals"
date:   2021-10-26 12:00:00
published: true
---

# Spawning processes without a supervision tree

# Single GenServer without Process.flag(:trap_exit, true)

## GenServer stopped with GenServer.stop


https://crypt.codemancers.com/posts/2016-01-24-understanding-exit-signals-in-erlang-slash-elixir/

Supervisors do not need to be explicitly stopped
https://hexdocs.pm/elixir/1.12/Supervisor.html#start_link/2

> Note that a supervisor started with this function is linked to the parent process and exits not only on crashes but also > if the parent process exits with `:normal` reason.

GenServers will also exit if the parent terminates with :normal if trapping exits:
https://hexdocs.pm/elixir/1.12/GenServer.html#start_link/3

> Note that a `GenServer` started with `start_link/3` is linked to the
> parent process and will exit in case of crashes from the parent. The GenServer
> will also exit due to the `:normal` reasons in case it is configured to trap
> exits in the `init/1` callback.


# tl;dr

Try to have processes under a supervision tree when designing and Elixir application, otherwise you will
need to write more code and be open to mistakes.

# Recap

1. :normal exit signals are ignored by the receiving process if not trapping exits.  If trapping exits they are received as messages.

unless trapping exits, in which case, these will be received as messages. If it’s sent by self, and not trapping exits, it will cause the process to terminate with a :normal exit reason, but will log an error.

2. :kill exit signals always result in the termination of the receiving process.

3. Exit signals with other reasons will terminate the receiving process unless trapping exits, in which case, these will be received as messages.


