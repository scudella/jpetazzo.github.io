---
layout: post
title: From dotCloud to Docker
---

Have you heard about dotCloud? If you haven't, I'm going to give you
a hint: it is a PAAS company. Another hint: eventually, dotCloud
open-sourced their container engine. That container engine became
Docker.

This is a quasi-archeological account of some of the early design
decisions of dotCloud, some of which have shaped how Docker is today
(and how it is not). "How is this relevant to my interests?" you ask.
If you are not using containers, and not planning to, ever, then
this article will not be very useful to you. Otherwise, I hope
that you can learn a lot from our past successes and failures.
At the very least, you will understand why Docker was built
this way.

*This was initially published as a [guest post] on [Taos]'
blog. I would like to thank [Julie Gunderson] for inviting
me to share this with Taos' audience!*

*Also, if you want to know more about the early story
of dotCloud and Docker, you should read [5 years at Docker]
by my coworker [Ken Cochrane]. It's pretty dope and
covers lots of things that I didn't mention there.*

[guest post]: http://www.taos.com/from-dotcloud-to-docker/
[Taos]: https://www.taos.com/
[Julie Gunderson]: https://twitter.com/Julie_Gund
[5 years at Docker]: https://www.kencochrane.net/2017/03/24/5-years-at-docker/
[Ken Cochrane]: https://twitter.com/KenCochrane


## First of all, a disclaimer

Don't consider this as a set of guidelines,
recommendations, or whatever. It's important to keep in mind
that when dotCloud was created (and for quite a while!),
things were very different:

- EC2 had just rolled out support for custom kernels
- EBS had major outages at least once a year
- Linux 3.0 wasn't out yet
- Consul and etcd didn't exist
- Go 1.0 hadn't been released yet

This might help to get some perspective on some of our technical
choices.

Take everything I say here with a grain of salt. I no longer have
access to the original dotCloud code, and while I knew that codebase
pretty well, I don't have an [eidetic memory](
https://en.wikipedia.org/wiki/Eidetic_memory) and it's very possible
(and even likely) that I misremember a few things. If you were
there, and think that I got something wrong, let me know! I'll be
happy to fix it.


## And then, a short page of boring history

*At `$STARTUP_NAME`, we always knew that containers were
the future, and we were using them before they were cool!
We are true containhipsters and we are glad that everybody
else is finally seeing the blinding light that we saw decades ago!*

This is not a cheap shot at [Bryan Cantrill](
https://twitter.com/bcantrill), for whom I have an inordinate
amount of respect and admiration. Sometimes I wish I had been
born earlier (and also smarter), and got a chance to work
on Solaris. Now *that's* a decent OS (even if the userland
is extremely picky about who it makes friends with), and
when the [Joyent crew runs containers](
https://docs.joyent.com/public-cloud/api-access/docker),
they're not messing around. (If you want a taste of
what it's like to run containers in production like a boss,
check out [this talk](https://www.youtube.com/watch?v=sYQ8j02wbCY),
you'll see what I'm talking about.)

But no Solaris for me! Instead, a friend whose hair and beard
could rival with Stallman's gave me a Slackware CD in the mid-90s,
and I've been stuck with Linux ever since. (I tried FreeBSD
once. I managed to crash the installer and then went on
to file one of the [most inane bug reports ever](
https://docs.freebsd.org/cgi/getmsg.cgi?fetch=0+0+archive/2000/freebsd-bugs/20000116.freebsd-bugs).)

Fast forward to 2008, when fellow hacker Solomon Hykes
gives me (and others) a demo of dotCloud. Back then, dotCloud
was a CLI tool allowing to author container images, move
them around, and easily instantiate them on multiple machines.
That demo was honestly pretty similar to the one in
[Solomon's lightning talk at PyCon in 2013](
https://www.youtube.com/watch?v=wW9CAH9nSLs),
but the tech behind it was very different. And for a good reason:
what Solomon demo'ed in 2013 was the result of 5+ years of
trial, error, and learning hard truths the hard way.


## Flintstone's Docker

![bricks](/assets/bricks.jpg)
*This is what our first containers looked like*

The dotCloud container engine (the ancestor of Docker) started as a Python CLI tool
called `dc`. (Yes, we knew that it conflicted with the old-school desk calculator
program. No, we didn't care.)

`dc` acted as a frontend to LXC and AUFS. Specifically, `dc` could:

- manage container images (pull/push them from/to a registry),
- create a container using one of these images (leveraging AUFS copy-on-write),
- configure the container, by allowing any file to be generated from a
  template (for instance, putting the correct IP address in /etc/network/interfaces),
- start the container, by automatically creating its LXC configuration
  file and invoking `lxc-start`,
- dynamically expose ports, by managing a set of `iptables` rules,
- and a few other cool things.

This is what interacting with `dc` looked like.
Keep in mind that I haven't used `dc` in 3 years and I don't have the
code anymore, so this is only approximate.

```bash
# pull an image
dc image_pull ubuntu@f00db33f
# create a container
dc container_create ubuntu@f00db33f jerome.awesome.ubuntu
# start the container
dc container_start jerome.awesome.ubuntu
# enter the container, like with "docker exec"
dc container_enter jerome.awesome.ubuntu
# there are a bunch of commands to manage port mappings
# the following one will allocate a random port
dc container_connection_add jerome.awesome.ubuntu tcp 80
# check which port was allocated
dc network_ls
```

So far, so good.

There are a few profound differences, however, between `dc` and modern Docker.


### Image format

We wanted to be able to track and audit accurately changes made to containers,
and possibly "transplant" them (e.g. when a new release of Ubuntu comes out,
run your application on that new release without rebuilding it.) It sounds like
a good idea at first! We stored images in Mercurial repositories, using the
`metashelf` extension to track special files and permissions. This means that
images operations were *slow*. Furthermore, it turns out that you can't
"rebase" a filesystem image like you would rebase a bunch of source commits.
It *kind of works* as long as you're only changing configuration files;
but it's useless if these configuration files are generated from templates
anyway. And it *doesn't work at all* for binaries or bytecode.

As a result, authoring images was a slow, bulky process, requiring some
extra tooling to be done efficiently. One of us ([Louis Opter](
https://twitter.com/1opter) if I remember correctly) was generally
in charge of updating the dotCloud official images; and everybody else
hoped that they'd never have to do it.

That's why Docker just used AUFS layers as-is. It was good enough,
it was fast, and since an AUFS layer is just a bunch of files masking
their counterparts in the original image, it means that you can still
get the list of modifications very easily.


### Dependency on AUFS

The dotCloud platform ran for about 5 years, and used AUFS all along. I'm
going to be brutally honest: no other option would have worked for us.
BTRFS used too much memory (and still does), because multiple containers
running the same image lead to page cache duplication. Device Mapper
thin provisioning didn't exist (and has the same memory issues anyway).
Ditto for ZFS. The other union filesystems (UnionMount, anyone?) were
hardly maintained, and had *tons* of edge cases.

That's why we used AUFS. It had its quirks, but *for our use case*,
it worked beautifully. It allowed us to
pack hundreds of containers on instances with 32 GB of RAM. It ran
flawlessly everything we threw at it, except MongoDB (something to do
with fancy mmap semantics), which prompted us to introduce volumes.

When we rolled out the first versions of Docker, we *knew* that the
dependency on AUFS would eventually become an issue. We were
particularly lucky, in the sense that it became an issue *after*
Docker got enough traction to convince Red Hat to do a lot of the
hard work involved to bring Docker to mainstream kernels.
That's how [Alexander Larsson](https://github.com/alexlarsson)
(and later, [Vincent Batts](https://github.com/vbatts)) ended up
writing the Device Mapper and BTRFS "graph drivers." On the
Docker side, core maintainers [Guillaume Charmes](
https://twitter.com/charme_g),
[Michael Crosby](https://twitter.com/crosbymichael), [Victor Vieux](
https://twitter.com/crosbymichael) and [Solomon Hykes](
https://twitter.com/solomonstre) himself did the heavy lifting
required to modularize that part of the Docker Engine.


### Entering a running container

Back in the day, you couldn't do the equivalent of `docker exec`
or `nsenter`, because they both rely on the `setns()` system call.
That system call appeared in Linux 3.0, in 2011. So how did we do,
then?

If you have used LXC, you might remember `lxc-attach`, which gives
you a console on a running container. It *could have worked*,
but we found it rather capricious. It was acceptable if you just
wanted to get a terminal in a runaway container; but you couldn't
depend on it as a remote command execution engine to setup
database replication, for instance. It was conceptually closer
to a serial console.

This leaves you with two options:

- patch your kernel to add support for the `setns()` system call;
- run an SSH server in your containers.

We did both. We had an abstract execution engine that would use
`setns()` when available, and fallback to SSH otherwise.

This means that our containers were all running an SSH server.
I was a huge fan of this SSH server, by the way, because it
allowed me to do all kinds of cool hacks. This
may come as a surprise, especially when one knows that I wrote
[this blog post](http://jpetazzo.github.io/2014/06/23/docker-ssh-considered-evil/),
but that merely demonstrates my ability to change my mind, amirite?

Why have both? Because we wanted the performance and convenience
of `setns()`, but we didn't want to rely on it and be forced to
stick to an older kernel if a wild kernel vulnerability appeared.


### The container daemon itself

Since containers are managed by LXC, you don't need a long-running daemon
(and at this point there was no container engine per se). In fact, if you
scratch the surface, you realize that each container has its own long-running
daemon: it's `lxc-start` (it's similar to `rkt` or `runc`) and you connect
to it using an abstract socket (from memory, `@/var/lib/lxc/<containername>`).

This is great, because it's simple. At least, it *seems* simple.
Each container was fully contained (so to speak) within
`/var/lib/dotcloud/<containername>`, so you could move a container
simply by copying that directory to another machine. Of course,
copying this directory *while the container is running* requires
extra precautions; but there was something satisfying and UNIX-y
in the fact that a container was just a directory, after all.


## Of course we couldn't help but build our own RPC layer

Perfect, we have our `dc` tool on our container nodes; now we need
to slap an API on top of that to orchestrate
deployments from a central place. Since containers are standalone, the
process exposing that API doesn't have to be bullet-proof, and you can
update/upgrade/restart it without being worried about your containers
being restarted.

Almost all the communication between processes and hosts was done
using [ZeroRPC](http://www.zerorpc.io/). ZeroRPC
is basically RPC over ZeroMQ, using MessagePack to serialize parameters,
return values, and exceptions. MessagePack is similar to JSON, but way
more efficient. (We didn't care much about efficiency except for the
high-traffic use cases like metrics and logs.)

If you're curious about ZeroRPC, [I presented it at
PyCon](https://www.youtube.com/watch?v=9G6-GksU7Ko) a few years ago.
Unfortunately, my French accent was a few orders of magnitude thicker than it
is today (which says a lot) so you might struggle to understand me, sorry ☹

ZeroRPC allowed us to expose almost any Python module or class like this:

```bash
# Expose Python built-in module "time" over port 1234
zerorpc-server --listen tcp://0.0.0.0:1234 time &
# Call time.sleep(4)
zerorpc-client tcp://localhost:1234 sleep 4
```

ZeroRPC also supports some fan-out topologies, including broadcast (all nodes
receiving the function call; return value is discarded) and worker queue
(all nodes subscribe to a "hub;" you send a function call to the hub,
one idle worker will get it, so you get transparent load balancing of requests).

The original ZeroRPC was synchronous, but [François-Xavier Bourlet](
https://github.com/bombela) implemented an asynchronous version (making use
of coroutines), as well as "streaming" — basically, the ability for a function
to return an iterator/generator, very useful for logs and live metrics!
[Andrea Luzzardi](https://github.com/aluzzardi) also implemented the zerotracer,
which allowed us to get full traces of API calls using transparent middlewares.
But I digress.


## Let's sprinkle micro-services all over

So here we are, with a "containers" service running on each node,
letting us do the following operations from a central place:

- create containers
- start/stop/destroy them

Listing containers (and gathering core host metrics) relied on a separate
service called "hostinfo." This service would just scan all the containers
deployed locally, aggregate their satus, and send it all to a central
place.

So thanks to "hostinfo" we can also list all containers from that central place.
Cool.

In the very first versions, dotCloud was building your apps "in place," i.e. when
you push your code, the code would be copied to a temporary directory in the container
(while it's still running the previous version of your app!), the build would happen,
then a switcheroo happens (a symlink is updated to point to the new version) and
processes are restarted.

To keep things clean and simple, this build system was managed by a separate service,
that directly accessed the container data structures on disk. So we had the "container manager,"
"hostinfo," and the "build manager," all accessing a bunch of containers and
configuration files in the same directory (`/var/lib/dotcloud`).

Then we added support for separate builds (probably similar to Heroku's "slugs").
The build would happen in a separate container; then that container image would
be transferred to the right host, and a switcheroo would happen (the old container is
replaced by the new one).

We had the equivalent of volumes, so by making sure that the old and new containers
were on the same host, this process could be used for stateful apps
as well. This, by the way, was probably a Very Bad Idea; as ditching away stateful
apps would have simplified things immensely for us. Keep in mind, though, that
we were running not only web apps but also databases like MySQL, PostgreSQL, MongoDB,
Redis, etc. I was one of the strong proponents of keeping stateful containers on board,
and on retrospect I was very certainly wrong, since it made our lives way more
complicated than they could have been. But I digress again!


### Who would have guessed that sharing state through on-disk files was a bad idea?

To keep things simple and reduce impact to existing systems (at this point, we had
a bunch of customers that each already generated more than $1K of monthly revenue,
and we wanted to play safe), when we rolled out that new system, it was managed
by another service. So now on our hosts we had the "container manager," "hostinfo,"
the "build manager" (for in-place builds), and the "deploy manager."

(Small parenthesis: we didn't transfer full container images, of course. We
transferred only the AUFS `rw` layer; so that's the equivalent of a two-line
Dockerfile doing `FROM python-nginx-uwsgi` and `RUN dotcloud-build.sh` then
pushing the resulting image around.)

Then we added a few extra services also accessing container data; in no specific
order, there was a remote execution manager (used e.g. by the MySQL replication
system), a metrics collector, and a bunch of hacks to work around EC2/EBS issues,
kernel issues, out of memory killer, etc.; for instance in some scenarios,
the OOM killer would leave the container in a weird state and we would need a few
special operations to clean it up. In the early day this was manual ops work,
but as soon as we had enough data it was automated.

The process tree on a container node looked like this:

```
- init -+- container
        +- hostinfo
        +- runner
        +- builder
        +- deployer
        +- metrics
        +- oomwrangler
        +- someotherstuff
        +- lxc-start for container X -+- process of container X
        |                             \- other process of container X
        +- lxc-start for container Y --- process of container Y
        \- lxc-start for container Z --- process of container Z
```

So at this point we have a bunch of services accessing a bunch of on-disk
structures. Locking was key. The problem is, that some operations are slow,
so you don't want to lock when unnecessary (e.g. you don't want to lock
everything while you're merely pulling an image). Some operations can
fail gracefully (e.g. it's OK if metrics collection fails for a few minutes).
Some operations are really important and you absolutely want to know if
they went wrong (e.g. the stuff that watches over MySQL replica
provisioning). Sometimes it's OK to ignore a container for a bit (e.g. for
metrics) but sometimes you absolutely want to know if it's there (because
if it's not, a failover mechanism will spin it up somewhere else; so having
containers disappearing in a transient manner would be bad).

To spice things further up, our ops toolkit was based on the `dc` CLI tool,
so that tool had to play nice with everything else.

Still with us? Get ready for another episode of "embarrassing early
start-up decisions."


## Dalek says, ORCHESTRATE

When your container platform runs 10 containers on a handful of nodes,
you can place them manually; especially if you don't create or resize
containers all the time.

But when you have thousands of containers (dotCloud peaked above 100,000
containers) running across hundreds of nodes, and your users constantly
deploy and scale services, you need an orchestrator. More specifically,
you need to automate *resource scheduling*.

In dotCloud's case, we wanted to be able to make an API call to create
a container, with the following parameters:

- some unique identifier for this container (combination of owner, app,
  service...)
- container base image and some templating parameters
- resources needed (mainly RAM)
- a "high availability token" which was used to prevent two containers
  with the same token from running on the same host (e.g. the leader
  and follower of a replicated database)

The API call should pick a machine to run the container, while honoring
the various constraints specified in the call (available resources and
HA token).


### And by "orchestration" I mean "scheduling"

In theory, any good CS grad student will tell you that this seems like a
perfectly good case to use some bin packing algorithm.

In practice, anybody who has worn a pager long enough knows that
network latency and packet loss are both non-zero quantities, and
that therefore, we are facing a distributed systems problem (aka
potential nuclear waste dumpster fire).

Most "standard" algorithms assume that you know the full state of
the cluster when taking a scheduling decision. But in our scenario,
you don't know the state of the cluster. You have to query each
machine. The request has to go over the network, and then the machine
has to read the state of all its containers before replying.
Both operations (network round-trip and gathering container state)
can and will take some time. Using aggressive timeouts (to avoid
waiting forever for unreachable nodes) gets problematic when
a host is very loaded (and takes a while to gather container state).

"Caching," I heard someone say. Excellent idea! Caching is easy.
*Cache invalidation,* however, is one of the hardest things
in computer science. How, why, when would we have to invalidate
the cache? Whenever some other system (other than the scheduler)
makes changes to the containers. We ended up having lots of
cluster maintenance tasks to move containers around, resize them,
regroup them (if you regroup containers using the same image,
you realize *huge* memory savings). These operations were
implemented by relatively simple scripts, relying on the
fact that each container was fully contained in a directory.
To preserve these semantics, you need to somehow watch
all your container configurations for changes, and trigger
cache invalidation events when changes happen. Alternative
option: implement these operations with the scheduler.
That was not realistic. We were constantly expecting the
unexpected (AWS "degraded performance" and "elevated error rates,"
some customer spiking 100x, 1 Mpps distributed denial-of-service
attacks, etc.) so we needed the ability to cobble solutions
*fast*, without breaking too many things. Central scheduler was
out, at least in the beginning.


### Distributed, robust, suboptimal scheduling

Our scheduler would just broadcast the request to a subset
of nodes, and place it in a retry queue.
These nodes would try to acquire a lock (implemented
by a centralized Redis). The one acquiring the lock would
carry on and deploy your container. Once the container is
up and running, another system takes care of removing it
from the retry queue (which gets re-broadcasted once in
a while).

That's an extremely naive algorithm, but it's also very
resilient. By the way, some of the fastest search algorithms
work by scattering your request on multiple nodes and
gathering only the first (fastest) replies, to make sure
that you get a good response time (at the expense of
correctness if some nodes are overloaded or down).

The main single point of failure was the Redis used for locking, and even
that one was not a big deal since it didn't really
*store* anything. (We always meant to replace it with
Zookeeper or Doozer, but it never turned out to be worth
the pain!)

If this "algorithm" makes you cringe, that's fair.
We weren't particularly proud of it, and we wanted
our next container engine to support better semantics.


## Summoning daemons

At this point, we really *dreamed* of a single point of entry to the
container engine, to avoid locking issues. At the very least, all
container metadata should be mediated by an engine exposing a clean API.
We had a pretty good idea of what was needed, and that's what shaped
the first versions of the Docker API.

The first versions of Docker were still relying on LXC. The
process tree on the container nodes would have looked like this:

```
- init -+- container
        +- hostinfo
        +- runner
        +- builder
        +- deployer
        +- metrics
        +- oomwrangler
        +- someotherstuff
        +- docker
        +- lxc-start for container X -+- process of container X
        |                             \- other process of container X
        +- lxc-start for container Y --- process of container Y
        \- lxc-start for container Z --- process of container Z
```

"Waitaminute," you say, "that's exactly the same thing as before!"

Yes! But now, all our management processes (container, hostinfo, etc.)
would go through "docker" instead of accessing container metadata
on disk. No more crazy locking, no more hoping that everybody used
the locking primitives correctly instead of accessing stuff
directly, etc.


## From LXC to libcontainer

Then, as containers picked up steam, LXC development (which was pretty
much dead, or at least making very slow progress) came to life,
and in a few months, there were more LXC versions than in the few years before.
This broke Docker a few times, and that's what led to the development of
libcontainer, allowing to directly program cgroups and namespaces without
going through LXC. You could put container processes directly under the
container engine, but having an intermediary process helps *a lot*,
so that's what we did; it was named `dockerinit`.

The process tree now looked like this:

```
- init --- docker -+- dockerinit for container X -+- process of container X
                   |                              \- other process of container X
                   +- dockerinit for container Y --- process of container Y
                   \- dockerinit for container Z --- process of container Z
```

But now you have a problem: if the docker process is restarted, you
end up orphaning all your "dockerinits." For simplicity, docker and dockerinit
share a bunch of file descriptors (giving access to the container's stdout
and stderr). The idea was to eventually make dockerinit a full-blown, standalone
mini-daemon, allowing to pass FDs around across UNIX sockets, buffering logs,
whatever's needed.

Having a daemon to manage the containers (we're talking low-level management
here, i.e. listing, starting, getting basic metrics) is crucial.
I'm sorry if I failed to convince you that it was important; but believe me,
you don't want to operate containers at scale without some kind of API.
(Executing commands over SSH is fine until you have more than 10 containers
per machine, then you really want a true API ☺)


## When the daemon does too much

But at the same time, the Docker Engine has lots of features and
complexity: builds, image management, semantic REST API over HTTP, etc.;
those features are essential (they are what helped to drive container
adoption, while vserver, openvz, jails, zones, LXC, etc. kept containers
contained (sorry!) to the hosting world) but it's totally reasonable
that you don't want all that code near your production stuff.

That's why in Docker Engine 1.11, we decided to break away
all the low-level container management functions to containerd,
while the rest would stay in the Docker Engine.

So the current solution is to delegate all the low-level management
to [containerd](https://containerd.io/), and keep the rest in the Docker Engine.

You can think of containerd like a simplified Docker Engine. You can
do the equivalent of `docker ps`, `docker run`, `docker kill`; but it
doesn't deal with builds, and its API uses grpc instead of REST.

The process tree looks like this:

```
- init - docker - containerd -+- shim for container X -+- process of container X
                              |                        \- other process of container X
                              +- shim for container Y --- process of container Y
                              \- shim for container Z --- process of container Z
```

The big upside (which doesn't appear on the diagram) is that the link
between docker and containerd can be severed and reestablished, e.g. to
restart or upgrade the Docker Engine. (This can be achieved with the
[live restore](https://docs.docker.com/engine/admin/live-restore/)
configuration option.)


## Going full circle

The dotCloud container engine started as a simple, standalone CLI tool.
It was augmented with a collection of "sidekick" daemons, each providing
a little bit of extra functionality. Eventually, this architecture showed
its limits. The first version of the Docker Engine gathered all the
features that were deemed necessary in a single daemon.
Too many features? I don't think so; precisely because these
features made the success of Docker. The first versions
of Docker sacrificed modularity, but that was only temporary.
Over time, features were separated from the Docker Engine again.
Today, you can use [runc](https://runc.io/) or [containerd](https://containerd.io/)
to run containers without the Docker Engine. Clustering features
are provided by [SwarmKit](https://github.com/docker/swarmkit).
External image builders are available, e.g. [dockramp](https://github.com/jlhawn/dockramp)
or [box](https://erikh.github.io/box/).

Two years ago, Docker committed to the motto "batteries included, but
swappable." It's still doing exactly that: providing what most people
need to build, ship, and run containerized apps, but giving an
increasing number of options to remove whatever you don't need or
don't like. And it all started with some really, really embarrassing
container management code in Python, almost 10 years ago!


