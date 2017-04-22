---
layout: post
title:  "Observing remote Elixir Docker nodes"
date:   2017-04-22 12:54:57 +0100
published: true
---

# Observing remote Elixir Docker nodes

Two good blog posts by [Martin Feckie](http://mfeckie.github.io/Remote-Profiling-Elixir-Over-SSH/) and another by [Erich Kist](http://blog.plataformatec.com.br/2016/05/tracing-and-observing-your-remote-node/)] document how to to connect to a remote Elixir node from your local machine in order to connect a remote iex session or run the Observer.  However if your Elixir (or Erlang) application is running in a Docker container on the remote host this is more complicated.

# What is EPMD?

EPMD (Erlang Port Mapper Daemon), is a part of the Erlang runtime system, written in C, that acts as a name server for distributed Erlang. When an Erlang node starts in distributed mode (by setting the `-name` parameter on startup) it checks to see if there is already an EPMD instance running bound to the loopback address ( and by default listening on port 4369), and if not starts one.  It then chooses a random port for inter-node communication, and registers its name and corresponding port with EPMD.  You can also start EPMD manually in the foreground which is useful for debugging:

```bash
killall epmd # Ensure any running daemon instance is stopped
epmd -d -d # Putting -d twice gives more debugging information
```

Normally each server will have its own EPMD instance, but each EPMD instance can have multiple Erlang nodes on that server registered to it.

When you connect to a remote node using the Elixir function `Node.connect :'node1@123.45.67.89'` it will attempt to connect to the EPMD listening on 123.45.67.89 port 4369, which will tell the calling instance the port that `node1` has allocated itself.  The local and remote nodes can then communicate.

The approach discussed in Martin Feckie and Erich Kist's blog articles involve creating an SSH tunnel to the remote server and then using the remote EPMD instance to register the local node, so that there is only a remote EPMD instance and no local one.

# Why this approach does not work with Docker

EPMD only allows nodes on the localhost to register themselves - requests from remote IPs will be rejected.  From the [EPMD documentation](http://erlang.org/doc/man/epmd.html) :

> It is always an error to try to register a node name if the client is not a process on the same host as the epmd
> instance is running on. Such requests are considered hostile and the connection is closed immediately.

The SSH tunnel makes it appear that the local node is on the same host, so this works when EPMD is running directly on the remote server (and not in a Docker container on the server).

I found this out by starting a container on a remote host and starting EPMD in debug mode in the container:

```bash
me@myserver.com:~$ docker run --rm -it elixir:1.4.2-slim bash
root@049fdf67639c:/# hostname -i
172.17.0.2
root@049fdf67639c:/# epmd -d -d
epmd: Sat Apr 22 19:22:41 2017: epmd running - daemon = 0
epmd: Sat Apr 22 19:22:41 2017: try to initiate listening port 4369
epmd: Sat Apr 22 19:22:41 2017: entering the main select() loop
```

Next I ensured there was no running local EPMD instance and created an SSH tunnel from my laptop to the remote server to map port 4369 locally to 172.17.0.2:4369
```bash
my-laptop$ killall epmd
my-laptop$ ssh me@myserver.com -L4369:172.17.0.2:4369 -N
```
Then I attempted to start a iex session locally (in a separate shell)
```bash
my-laptop$ iex --name node@my-laptop --cookie mycookie
Protocol 'inet_tcp': register/listen error: epmd_close
```

The request to register in EPMD fails and looking at the stdout in the container we can see that it is rejected as it is a non-local node and the connection is closed:
```bash
epmd: Sat Apr 22 19:32:41 2017: Non-local peer connected
epmd: Sat Apr 22 19:32:41 2017: opening connection on file descriptor 5
epmd: Sat Apr 22 19:32:41 2017: got 19 bytes
***** 00000000  00 11 78 d1 05 4d 00 00  05 00 05 00 04 6e 6f 64  |..x..M.......nod|
***** 00000010  65 00 00                                          |e..|
epmd: Sat Apr 22 19:32:41 2017: ** got ALIVE2_REQ
epmd: Sat Apr 22 19:32:41 2017: ALIVE2_REQ from non local address
epmd: Sat Apr 22 19:32:41 2017: closing connection on file descriptor 5
```

# Solution

Instead of trying to get both the local node and remote node to use the same EPMD instance, we will create two separate EPMD instances and then use an SSH tunnel so that we can connect to the remote EPMD instance and remote Erlang node locally.  However, as all EPMD instances in a cluster must run on the same port (as per the EPMD documentation), and our local instance will be bound to 127.0.0.1 port 4369 we have to tunnel the remote instance to a local IP.

We could use the eth0/en0 IP address, but as this can change, I choose to create a 2nd loopback address.  On a Mac you can do this as follows:
```bash
my-laptop$ sudo ifconfig lo0 alias 127.0.0.2
```
(To remove it again do `sudo ifconfig lo0 -alias 127.0.0.2`  - note the hyphen)

We then start the remote Elixir instance in the container.  We need to set the cookie and also the node name (the IP of the name must match the additional loopback IP we have created as this is how we will connect to it locally).

We also use the `inet_dist_listen_min/max` parameters to ensure that node listens on port 19000 so we can forward this port.

```bash
me@myserver.com:~$ docker run --rm -it elixir:1.4.2-slim bash
root@866b1839d4fe:/# hostname -i
172.17.0.2
root@866b1839d4fe:/# iex --cookie mycookie --name remotenode@127.0.0.2 --erl "-kernel inet_dist_listen_min 19000 inet_dist_listen_max 19000"
Erlang/OTP 19 [erts-8.3.1] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false]

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(remotenode@866b1839d4fe)1>
```

Now we create the SSH tunnel and map the remote container EPMD port and port 19000 onto the extra loopback address.
```bash
my-laptop$ ssh me@myserver.com -L127.0.0.2:4369:172.17.0.2:4369 -L127.0.0.2:19000:172.17.0.2:19000 -N
```

...and we start the local iex session.  EPMD will attempt to bind to 127.0.0.1 and all other available IPs, but as 127.0.0.2 is already bound via SSH it will not bind to this.
```bash
my-laptop$ iex --name node@my-laptop --cookie mycookie
Erlang/OTP 19 [erts-8.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(node@my-laptop)1>
```

So now we have two EPMD instances running on port 4369:
1. Remotely, in the remote container 172.17.0.2, and also tunnelled locally to 127.0.0.2
2. Locally on 127.0.0.1

Now we can attempt to connect to the remote node from our local iex session:
```
iex(node@my-laptop)1> Node.connect :'remotenode@127.0.0.2'
true
```

Now we start Observer (`:observer.start`) and then we should see our remote node in the Nodes menu of Observer and we can connect to it.

# Conclusion

This is quite a complex setup, but it does provide what is required. I do not particularly like having to create an additional loopback adapter and also having to specify the additional loopback IP as part of the name of the remote node.

I plan to research doing this in another way by creating a reverse SSH tunnel to make the local EPMD accessible in another container on the remote server, and then running `Node.connect/1` from one container to the other.  That way an extra loopback IP should not be required.
