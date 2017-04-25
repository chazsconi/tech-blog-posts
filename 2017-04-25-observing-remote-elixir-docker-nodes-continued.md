---
layout: post
title:  "Observing remote Elixir Docker nodes - Continued"
date:   2017-04-25 22:54:57 +0100
published: true
---

# Previous findings
In [a previous blog post]({% post_url 2017-04-22-observing-remote-elixir-docker-nodes %}) post I showed how to connect to a remote Elixir node running inside a Docker container.  However the solution is a bit ugly as it involves creating an additional loopback adapter on your local machine.  Here I believe is a better solution that avoids this.

# Better solution

The idea here is instead of forwarding the server's ports to the local machine, we forward the local machine's ports to the server and then access this from within our running Elixir Docker container.

## Enable GatewayPorts
By default remote port forwarding can only bind to the loopback adapter on the remote host which will not be accessible from a container.  To change this we must edit `/etc/ssh/sshd_config` on the server and add this line:
```
GatewayPorts clientspecified
```
Ensure there are no other `GatewayPorts` lines in the file.  Next, reload the configuration by executing `sudo service ssh restart`.  (These instructions are for Ubuntu, it might be slightly different on other distributions.)

## Run a remote IEX session in a container

We then start an Elixir node on the server in a Docker container.
```bash
me@myserver.com:~$ docker run --rm -it elixir:1.4.2-slim bash
root@866b1839d4fe:/# iex --cookie mycookie --name remotenode@127.0.0.1
Erlang/OTP 19 [erts-8.3.1] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:10] [hipe] [kernel-poll:false]

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(remotenode@127.0.0.1)1>
```

## Run a local IEX session

We also start a local IEX session.  We need to set the cookie and also the node name.  The IP of the name is the Docker bridge interface (docker0) on the server (by default `172.17.0.1`) as our remote node will use this to connect.

We also use the `inet_dist_listen_min/max` parameters to ensure that node listens on port 19000 so we can forward this port.

```bash
my-laptop$ iex --name node@172.17.0.1 --cookie mycookie --erl "-kernel inet_dist_listen_min 19000 inet_dist_listen_max 19000"
Erlang/OTP 19 [erts-8.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(node@172.17.0.1)1>
```
## Forward local ports to the server

Next we create the SSH tunnel and map the local EPMD port and port 19000 onto the Docker bridge IP on the remote server.

```bash
ssh me@myserver.com -R 172.17.0.1:4369:127.0.0.1:4369 -R 172.17.0.1:19000:127.0.0.1:19000 -N
```

## Connect the nodes together

Now we can get the remote node to connect to our local node:
```bash
iex(remotenode@myserver.com)1> Node.connect :'node@172.17.0.1'
true
```

The connection is bi-directional so on our local node we can see that our remote node is now connected and we can also start the Observer.  Once started, we can select our remote node from the Observer 'Nodes' menu.
```bash
iex(node@172.17.0.1)2> Node.list
[:"remotenode@myserver.com"]
iex(node@172.17.0.1)3> :observer.start
:ok
```
# What if there we do not have an IEX session on the remote node?

Normally our remote node will not be running an IEX session and will actually be running some application so we do not have the ability to just type the `Node.connect` in the prompt.

We can solve this by creating a temporary node inside our running container and then using this to send an RPC call to our running node to tell it to connect to our local node.

```bash
me@myserver.com:~$ docker exec 866b1839d4fe elixir --name temp@127.0.0.1 --cookie mycookie -e ":rpc.call(:'remotenode@127.0.0.1', Node, :connect, [:'node@172.17.0.1'])"
```
In the above example `866b1839d4fe` is the ID of our running remote container.

# Putting it all together

I have written a script to combine all these steps: 

[chazsconi/connect-remote-elixir-docker](https://github.com/chazsconi/connect-remote-elixir-docker)

# Conclusion

This solution is perhaps a bit more complex that the previous one, but it has several advantages
1. It does not involve creating another loopback adapter locally which needs later cleanup.
2. The remote node does not need to be configured with any extra Erlang configuration parameters (it just needs a name and a cookie)
3. If there are multiple nodes running on the remote server in different containers, they can all be connected to the local machine.

The only minor drawback is having to change the SSH configuration on the server initially.
