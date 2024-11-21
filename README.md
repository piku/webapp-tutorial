# Build a web app fast (with piku)

> This repository is a fork of [@amontalenti](https://github.com/amontalenti)'s [original lightning talk tutorial](https://github.com/amontalenti/webappfast-piku), and the `piku` project recognizes his authorship and ownership rights (as discussed in [#200](https://github.com/piku/piku/issues/200)). We are also very grateful for his contribution and sense of humor. 😄

## Quick start 🚀

The assumption here is that you have a development environment as follows:

- a local Linux or Mac machine/VM which has an `ssh` client installed
- a network-available homelab server, VPS, or cloud node with `root` access
- a local Python environment, e.g. managed with `pyenv` and/or `pipx`

Given this environment, let's say I want to use [piku][piku]. WIIFM? That is,
"What's In It For Me?"

[piku]: https://github.com/piku/piku#readme

The goal here is to have Heroku-style deploys, with none of the Heroku (or
dokku) style cognitive overhead. What is a Heroku-style deploy?

- You write code in a local git repo.
- You push to a "special" branch (in this case, called "piku") to deploy your
  code to a remote alpha/beta/staging environment, where your webapp is
automatically mounted and running.
- You can control aspects of that environment in a way that'll be easy to
  mirror in production, using source artifacts that are simple files like `ENV`
(for env variables) and `Procfile` (for defining your app's entrypoints).
- You can perform basic admin operations on your Heroku-like environment, such
  as viewing logs, destroying apps, restarting apps, and scaling apps (from 1
to N processes).

To get to this amazing end state, you'll need to start by bootstrapping a
remote machine with the `piku` infrastructure.

This remote machine can be a locally-available homelab, live in a VPS, or live
in the cloud. As a preliminary step, you might want to use a tool like ZeroTier
to establish a secure network link with that remote node (in lieu of a VPN or
reverse tunneling via SSH).

We'll refer to that machine as `machine` from here on out. Make sure you can
`ssh` in, both as yourself, and as `root`:

    $ ssh machine
    $ ssh root@machine

If you can't, make sure to get `~/.ssh/authorized_keys` set up in the right
way, perhaps leveraging `ssh-copy-id` as necessary.

Once you can, you'll want to log in as root and run the piku bootstrap:

    $ ssh root@machine
    root@machine:~# curl https://piku.github.io/get | sh
    ... lots of output ...
    ... eventual success ...
    root@machine:~#
    <CTRL+D EOF to quit>
    logout
    Connection to x.x.x.x closed.

You'll know piku is setup because you should now be able to ssh in with the
`piku` user, and this will result in the piku CLI being shown to you. Try it:

    $ ssh piku@machine
    Usage: piku.py [OPTIONS] COMMAND [ARGS]...

    The smallest PaaS you've ever seen

    Options:
    -h, --help  Show this message and exit.

    Commands:
    apps              List apps, e.g.: piku apps
    ...
    Connection to x.x.x.x closed.

Then, as a helper, you can install the piku CLI locally on your machine.
Here, I installed it in `~/opt/bin`, which is my own little directory for
scripts, but you might find it convenient to put it in `/usr/local/bin` on
many UNIX machines. Do this with a simple download from GitHub:

    $ cd ~/opt/bin
    $ wget https://raw.githubusercontent.com/piku/piku/master/piku
    $ chmod a+rx piku

We'll come back to that later, it's mainly for convenience.

Your next step is to link this git repo to your piku node as a "remote".
This is a one-liner:

    $ git remote add piku piku@machine:webappfast

... and, with this addition, we can push to origin (which is GitHub) or piku
(our ssh-accessible `machine`).

I find it's best to have your piku target set as the default upstream,
and then only push to GitHub as you like. This gives you the option to use
a git-based workflow "semi-locally" (with 2 copies, your machine and the piku
remote), and then do squashing and even (gasp!) force-pushes as necessary
to clean up history before pushing to GitHub (origin).

## What is this magic?

Once piku is set up, it wires together some technologies and UNIX hacks
quite elegantly. I'll walk through them in turn.

### How is `ssh piku@machine` running... a piku CLI?

This is a pretty esoteric trick. It turns out, `~/.ssh/authorized_keys` can
be customized to pick the starting directory and even starting/shell command
of any ssh session. What piku does is customize this to make the `piku` user
run `piku.py` (that is, the piku Python command-line tool) upon `ssh`.

This Python command is, then, quite sophisticated. It gets wrapped by the
local `piku` CLI you installed above, to basically run "remote" commands
on your box, following the Heroku API pretty closely.

For those keeping score, this trick, alone, along with a simple shell wrapper,
pretty much implements the entire concept of Python's `fab` (aka `Fabric` or
`fab-classic`) tool, but with the Python layer moved to _the other side of the
SSH connection_. This is sort of brilliant and mind-boggling. But, rather than
having to re-implement the SSH protocol, via `paramiko` and similar, inside the
fabric Python library, with this trick, we implement a "plain" CLI which gets
"bootstrapped" onto the remote node, and then we execute the CLI over a _plain
SSH connection_, thus leveraging the standard `ssh` client for all the
`client->server` communication.

This means the CLI is "just" a standard Python CLI tool, one that is expected
to be run "over" an ssh connection. Awesome!

Here is how that looks in the `.authorized_keys` file:

    command="FINGERPRINT=SHA256:GW/2zbt... NAME=default \
    /home/piku/piku.py \
    $SSH_ORIGINAL_COMMAND"\
    ,no-agent-forwarding\
    ,no-user-rc,\
    no-X11-forwarding,\
    no-port-forwarding \
    ssh-rsa AAAAB...

Here is how the "command" feature is described in ssh docs:

> Forces a command to be executed when this key is used for authentication.
> This is also called 'command restriction' or 'forced command'. The effect is
> to limit the privileges given to the key, and specifying this options is
> often important for implementing the principle of least privilege. Without
> this option, the key grants unlimited access as that user, including
> obtaining shell access.

Fascinating!

### How is the git remote string as simple as `piku@machine:webappfast`?

It seems very magical that when you do a `git push` using a remote described
simply as `piku@machine:webappfast` that git will "just know" to make a git
repo for you in the special directory `/home/piku/.piku/repos/webappfast` on
the remote server, and treat that git repo as the remote. What the heck is
going on here?

Well, when combined with the above trick, the `piku.py` command-line tool
implements support for something known as the "git pack protocol", simply
by virtue of supporting the subcommand `git-receive-pack`. This protocol
is described in the [pack-protocol.txt][git-pack-protocol] documentation,
but the fact that it works is quite surprising.

Basically, on a "normal" UNIX system, a `git push` over the `ssh` protocol
results in a receipt command on the other side of the ssh connection that is
akin to:

    env git-receive-pack

Where the `env` is a no-op shell environment. But, in our "restricted" forced
command environment with piku.py, the command being run is:

    /home/piku/piku.py git-receive-pack

And, of course, this _works_ because piku.py implements a subcommand called
`git-receive-pack`. The part after the `:` in the `git push` string, which
is our "piku app name", gets sent as an extra parameter here, so it's really:

    /home/piku/piku.py git-receive-pack webappfast

The code that implements this does a couple of things:

- creates a repo if necessary in /home/piku/.piku/repos/APP
- initializes the repo, if necessary
- makes a post-commit hook in the repo config; this will matter later
- shells out to `git-receive-pack` (with our new CWD) to actually receive the
  git data

Pretty amazingly fancy. By this trick, you push to `piku@machine:app` and you
get a managed git repo under `piku@machine:/home/piku/.piku/repos/app`, with
post-commit hooks already configured.

### What does the post-commit hook do?

To make it possible to configure the application, and get it up-and-running,
upon every git push, requires some serious git post-commit hook smarts. Of
course, all the rest of that is implemented in `piku.py` as well.

When you push code to the piku remote, the post-commit-hook runs. This hook
does a few things:

- checks out the pushed commit of code into `/home/piku/.piku/apps/APP`
- scans the source code to determine the "runtime", e.g. when it has
  `requirements.txt`, it's a Python runtime
- creates an "env" for the app in `/home/piku/.piku/envs/APP`; for Python, this
  uses `venv`
- installs dependencies (for Python, this is via requirements.txt and `pip`)
- scans the `ENV` file to set environmental variables, 12-factor style
- scans the `Procfile` file to determine which apps should be mounted, and how
- as necessary, configures uwsgi (process manager), acme (certificate manager),
  and nginx (web server) from there
- points the logs into `/home/piku/.piku/logs/APP`

What's great is that all of this can happen immediately upon `git push`, since
we're already "ssh'ed in" to the remote machine. There is no need for a fab or
ansible substrate.

At that point, the app should be running. But, the `piku` CLI gives you a few
management options from there.

### What does the remote filesystem look like?

I've added a little helper command to `Makefile` called `make show`, which
will showcase the remote filesystem. Here is how mine looks with a single
app configured, a simple Flask app using uwsgi to serve up web requests,
nginx as the web server, and acme for managing the Let's Encrypt HTTPS
certificate.

    $ make show
    piku run -- tree /home/piku/.piku -L 2
    Piku remote operator.
    Server: piku@machine
    App: webappfast

    /home/piku/.piku
    ├── acme
    │   └── webappfast -> /home/piku/.acme.sh/zero.black
    ├── apps
    │   └── webappfast/..
    ├── envs
    │   └── webappfast/..
    ├── logs
    │   └── webappfast/..
    ├── nginx
    │   ├── webappfast.conf
    │   ├── webappfast.crt
    │   ├── webappfast.key
    │   └── webappfast.sock
    ├── repos
    │   └── webappfast/..
    ├── uwsgi
    │   ├── uwsgi.ini
    │   ├── uwsgi.log
    │   ├── uwsgi-piku.pid
    │   └── uwsgi.sock
    ├── uwsgi-available
    │   └── webappfast_wsgi.1.ini
    └── uwsgi-enabled
        └── webappfast_wsgi.1.ini

### What makes this especially good vs e.g. rsync or fab?

In a way, `git` is the best possible deployment tool. When you use rsync or
fab, you often try to get some of the benefits of using git _with_ them, e.g.
by doing a clean local checkout before making a source tree tarball to send
over to the server, or before running an rsync command.

But, git already knows how to receive **only the deltas** between your local
repo and the remote repo.  So, it will always send **only what it needs to**
over the wire. And, once they get over the wire, by doing a clean checkout of
the source tree from the updated repo, you are **guaranteed to only checkout
version-controlled artifacts**. You also have a pretty easy time doing a
rollback: just push a past commit to the branch. Heck, you can even use git's
own `git revert` command to keep even your rollback versioned.

The combination of the `piku.py` tool as a "smart git receiver" and also as a
"smart git post-commit" hook means that you have arbitrary Python code hooked
in to the two "interface points" with your server: code push and deploy. The
fact that the command is also a "smart ssh substrate" means that you get
ssh-based management commands basically "for free". For example:

    piku restart
    piku stop
    piku destroy
    piku logs

Are all commands available that restart the service, stop the service, destroy
the service (cleaning up all the server configuration files and environments),
and give you a tail on the logs for the service. Under the hood, these are
just "plain" ssh commands talking to the remote `piku.py` file. How cool is
that?

### How does piku.py bootstrap itself?

Final bit of magic: this README glazes over the bootstrap step, but how does
that work under the hood?

Well, this, too, is some Pythonic magic. Most of the smarts here are outsourced
to `ansible`, a project that is itself written in Python and which is easy to get
installed on any remote server, whether via built-in packages (e.g. `apt`). The
bootstrap script installs ansible, and then runs some basic playbooks to get
Python, nginx, uwsgi, git, and other required packages up-and-running on the server.
It also uses ansible to set up the ssh tricks I document above into the `piku`
user and `/home/piku` home directory. Simple as that!

### piku mentions not just "wsgi" , but also "cron" and "worker"; what's that?

In an attempt to mirror the functionality of Heroku, and support webapp
long-running sidecars, daemons, and cron-style repeated tasks, the `Procfile`
format supported by piku supports more than just the `wsgi` directive.

Some [basic docs on the `Procfile` format can be found here][procfile], but
here is what you should know, in a nutshell:

- use `wsgi` for Django, Tornado, Flask, and similar Python web apps;
  obviously, these get managed by uwsgi and then mounted behind nginx
- use `worker` for daemons; these, it turns out, also get managed by uwsgi,
  since uwsgi can operate as a generic process manager, handling restarts,
logging, and multi-worker spawning with built-in functionality similar to the
`supervisord` tool that is often used alongside it
- use `cron` for repeated web apps; believe it or not, this _also_ gets managed
  by uwsgi, since uwsgi has [cron-style tasks][cron]

This therefore explains the cleverness of piku: it outsources most of the
"service management" functionality to `uwsgi`, and then outsources "edge HTTP"
concerns to `nginx` (HTTP) and acme (certificates). By outsourcing code
deployment to `git` and server management to `ssh`, there is no need for
frameworks beyond stock Python 3 and the UNIX shell.

[git-pack-protocol]: https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt
[procfile]: https://piku.github.io/configuration/procfile.html
[cron]: https://uwsgi-docs.readthedocs.io/en/latest/Cron.html

### Where are the containers? Where is the orchestrator?

Bwahahahaha. There are no containers here, my friend. There are no orchestrators.

Sweet, sweet UNIX is all you need, you see!

Though containers can be quite nice to create a binary deploy artifact, of
sorts, out of your entire environment, it's worth mentioning _how much simpler_
this is than running your app under a container. Here, your app is nothing more
than a directory of source code, done from a clean `git checkout` from a `git`
branch. Your "environment" is defined using Python venvs, which provide much
of the same benefit of containers, but at a fraction of the cognitive and machine
cost. They are much lighter weight and easier to reason about, and they don't
introduce any new networking, disk, or filesystem abstractions. Finally, though
you might think about some sort of "orchestrator" for your "containers", here
we have a "process manager" (uwsgi) for your "processes" (python).

The subprocesses are managed the UNIX way, and thus are introspectable in all
the usual ways. The communication with other tools (like nginx) is happening
via plain sockets. The logging is happening via plain text files. Everything is
where it should be.

As for rollbacks and state, it's true that this whole setup isn't saying much
about your database (e.g. Postgres, Redis, Elasticsearch), nor about your
outside-of-the-source-tree production state. But, let's be honest: containers
never really helped you with these issues in production, either. They were
usually, if not always, managed separately. So, arguably, piku is focusing
on the part of your deployment that can actually be managed for you in a
set-it-and-forget-it way. To get your databases working and managed well,
and supporting complex data migration and administration operations, you'll
need to rely on the usual suspects: ansible, terraform, fabric.
