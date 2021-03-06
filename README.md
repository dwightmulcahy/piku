![piku logo](./img/logo.png)

The tiniest Heroku/CloudFoundry-like PaaS you've ever seen.

`piku`, inspired by [dokku][dokku], allows you do `git push` deployments to your own servers.

[![asciicast](https://asciinema.org/a/Ar31IoTkzsZmWWvlJll6p7haS.svg)](https://asciinema.org/a/Ar31IoTkzsZmWWvlJll6p7haS)

### Documentation: [Procfile](docs/DESIGN.md#procfile-format) | [ENV](./docs/ENV.md) | [Examples](./examples/README.md)

## Using `piku`

`piku` supports a Heroku-like workflow, like so:

* Create a `git` SSH remote pointing to your `piku` server with the app name as repo name.
  `git remote add piku piku@yourserver:appname`.
* Push your code: `git push piku master`.
* `piku` determines the runtime and installs the dependencies for your app (building whatever's required).
   * For Python, it segregates each app's dependencies into a `virtualenv`.
   * For Go, it defines a separate `GOPATH` for each app.
   * For Node, it installs whatever is in `package.json` into `node_modules`.
   * For Java, it builds your app depending on either `pom.xml` or `build.gradle` file.
* It then looks at a [`Procfile` which is documented here](docs/DESIGN.md#procfile-format) and starts the relevant workers using [uWSGI][uwsgi] as a generic process manager.
* You can optionally also specify a `release` worker which is run once when the app is deployed.
* You can then remotely change application settings (`config:set`) or scale up/down worker processes (`ps:scale`).
* You can also bake application settings into a file called [`ENV` which is documented here](./docs/ENV.md).
* A `static` worker type, with the root path as the argument, can be used to deploy a gh-pages style static site.

## Install

To use `piku` you need a VPS, Raspberry Pi, or other server bootstrapped with `piku`'s requirements. You can use a single server to run multiple `piku` apps.

**Warning**: You should use a fresh server or VPS instance without anything important running on it already, as `piku-bootstrap` will make changes to configuration files, running services, etc.

Once you've got a fresh server, download the [piku-bootstrap](./piku-bootstrap) shell script onto your local machine and run it:

```shell
curl https://piku.github.io/get | sh
```

The first time it is run `piku-bootstrap` will install itself into `~/.piku-bootstrap` on your local machine and set up a virtualenv there with the dependencies it requires. It will only need to do this once.

The script will display a usage message and you can then bootstrap your server:

```shell
./piku-bootstrap root@yourserver.net
```

If you put the `piku-bootstrap` script on your `PATH` somewhere, you can use it again to provision other servers in the future.

See below for instructions on [installing other custom dependencies](#installing-other-dependencies) that your apps might need like a database etc.

### `piku` client

To make life easier you can also install the [piku](./piku) helper CLI. Install it into your path e.g. `~/bin` to run it from anywhere.

```shell
./piku-bootstrap install-cli ~/bin
```

This shell script makes working with `piku` remotes a bit simpler. If you have a git remote called `piku` in the current folder it will infer the remote server and app name and insert those into the remote piku commands. This allows you do execute commands like the following on your running remote app:

```shell
$ piku logs
$ piku config:set MYVAR=12
$ piku stop
$ piku deploy
$ piku destroy
$ piku # <- will show help for the remote app
```

You can pass flags through to the underlying SSH command, for example `-t` to run interactive commands remotely, and `-A` to proxy authentication credentials in order to do remote git pulls.

Here is an example of using the `-t` flag to obtain a `bash` shell in the app directory of one of your Piku apps:

```
$ piku -t run bash
Piku remote operator.
Server: piku@cloud.mccormickit.com
App: dashboard

piku@piku:~/.piku/apps/dashboard$ ls
data  ENV  index.html  package.json  package-lock.json  Procfile  server.wisp
```

Tip: If you put this `piku` script on your `PATH` you can use the `piku` command across multiple apps on your local.

### Installing other dependencies

`piku-bootstrap` uses Ansible internally and it comes with several extra built-in playbooks which you can use to bootstrap common components onto your `piku` server.

Use `piku-bootstrap list-playbooks` to show a list of built-in playbooks, and then to install one add it as an argument to the bootstrap command.

For example, to deploy `nodeenv` onto your server:

```shell
piku-bootstrap root@yourserver.net nodeenv.yml
```

You can also use `piku-bootstrap` to run your own Ansible playbooks like this:

```shell
piku-bootstrap root@yourserver.net ./myplaybook.yml
```

## Examples

You can find examples for deploying various kinds of apps into a `piku` server in the [Examples folder](./examples).

## Motivation

I kept finding myself wanting an Heroku/CloudFoundry-like way to deploy stuff on a few remote ARM boards and [my Raspberry Pi cluster][raspi-cluster], but since [dokku][dokku] didn't work on ARM at the time and even `docker` can be overkill sometimes, I decided to roll my own.

## Project Status/To Do:

This is currently being used for production deployments of [my website](https://taoofmac.com) and a few other projects of mine that run on Azure and other IaaS providers. Regardless, there is still room for improvement:

From the bottom up:

- [ ] Prebuilt Raspbian image with everything baked in
- [ ] `chroot`/namespace isolation (tentative)
- [ ] Relay commands to other nodes
- [ ] Proxy deployments to other nodes (build on one box, deploy to many)
- [ ] Support Clojure/Java deployments through `boot` or `lein`
- [ ] Sample Go app
- [ ] Support Go deployments (in progress)
- [ ] nginx SSL optimization/cypher suites, own certificates
- [ ] Review deployment messages
- [ ] WIP: Review docs/CLI command documentation (short descriptions done, need `help <cmd>` and better descriptions)
- [ ] Lua/WSAPI support
- [x] Support for Java Apps with maven/gradle (in progress through jwsgi, by @matrixjnr)
- [x] Django and Wisp examples (by @chr15m)
- [x] Project logo (by @chr15m)
- [x] Various release/deployment improvements (by @chr15m)
- [x] Support Node deployments (by @chr15m)
- [x] Let's Encrypt support (by @chr15m)
- [x] Allow setting `nginx` IP bindings in `ENV` file (`NGINX_IPV4_ADDRESS` and `NGINX_IPV6_ADDRESS`)
- [x] Cleanups to remove 2.7 syntax internally
- [x] Change to Python 3 runtime as default, with `PYTHON_VERSION = 2` as fallback
- [x] Run in Python 3 only
- [x] (experimental) REPL in `feature/repl`
- [x] Python 3 support through `PYTHON_VERSION = 3`
- [x] static URL mapping to arbitrary paths (hat tip to @carlosefr for `nginx` tuning)
- [x] remote CLI (requires `ssh -t`)
- [x] saner uWSGI logging
- [x] `gevent` activated when `UWSGI_GEVENT = <integer>`
- [x] enable CloudFlare ACL when `NGINX_CLOUDFLARE_ACL = True`
- [x] Autodetect SPDY/HTTPv2 support and activate it
- [x] Basic nginx SSL config with self-signed certificates and UNIX domain socket connection
- [x] nginx support - creates an nginx config file if `NGINX_SERVER_NAME` is defined
- [x] Testing with pre-packaged [uWSGI][uwsgi] versions on Debian Jessie (yes, it was painful)
- [x] Support barebones binary deployments
- [x] Complete installation instructions (see `INSTALL.md`, which also has a draft of Go installation steps)
- [x] Installation helper/SSH key setup
- [x] Worker scaling
- [x] Remote CLI commands for changing/viewing applied/live settings
- [x] Remote tailing of all logfiles for a single application
- [x] HTTP port selection (and per-app environment variables)
- [x] Sample Python app
- [X] `Procfile` support (`wsgi` and `worker` processes for now, `web` processes being tested)
- [x] Basic CLI commands to manage apps
- [x] `virtualenv` isolation
- [x] Support Python deployments
- [x] Repo creation upon first push
- [x] Basic understanding of [how `dokku` works](http://off-the-stack.moorman.nu/2013-11-23-how-dokku-works.html)

## Internals

This is an illustrated example of how `piku` works for a Python deployment:

![](img/piku.png)

## Supported Platforms

`piku` is intended to work in any POSIX-like environment where you have Python, [uWSGI][uwsgi] and SSH, i.e.: 
Linux, FreeBSD, [Cygwin][cygwin] and the [Windows Subsystem for Linux][wsl].

As a baseline, it began its development on an original, 256MB Rasbperry Pi Model B, and still runs reliably on it.

Since I have an ODROID-U2, [a bunch of Pi 2s][raspi-cluster] and a few more ARM boards on the way, it is often tested on a number of places where running `x64` binaries is unfeasible.

But there are already a few folk using `piku` on vanilla `x64` Linux without any issues whatsoever, so yes, you can use it as a micro-PaaS for 'real' stuff. Your mileage may vary.

## Supported Runtimes

`piku` currently supports deploying apps (and dependencies) written in Python, with Go, Clojure (Java) and Node (see [above](#project-statustodo)) in the works. But if it can be invoked from a shell, it can be run inside `piku`.

## FAQ

**Q:** Why `piku`?

**A:** Partly because it's supposed to run on a [Pi][pi], because it's Japanese onomatopeia for 'twitch' or 'jolt', and because I know the name will annoy some of my friends.

**Q:** Why Python/why not Go?

**A:** I actually thought about doing this in Go right off the bat, but [click][click] is so cool and I needed to have [uWSGI][uwsgi] running anyway, so I caved in. But I'm very likely to take something like [suture](https://github.com/thejerf/suture) and port this across, doing away with [uWSGI][uwsgi] altogether.

Go also (at the time) did not have a way to vendor dependencies that I was comfortable with, and that is also why Go support fell behind. Hopefully that will change soon.

**Q:** Does it run under Python 3?

**A:** Right now, it _only_ runs on Python 3, even though it can deploy apps written in both major versions. It began its development using 2.7 and using`click` for abstracting the simpler stuff, and I eventually switched over to 3.5 once it was supported in Debian Stretch and Raspbian since I wanted to make installing it on the Raspberry Pi as simple as possible.

**Q:** Why not just use `dokku`?

**A:** I used `dokku` daily for most of my personal stuff for a good while. But it relied on a number of `x64` containers that needed to be completely rebuilt for ARM, and when I decided I needed something like this (March 2016) that was barely possible - `docker` itself was not fully baked for ARM yet, and people were at the time trying to get `herokuish` and `buildstep` to build on ARM.

[click]: http://click.pocoo.org
[pi]: http://www.raspberrypi.org
[dokku]: https://github.com/dokku/dokku
[raspi-cluster]: https://github.com/rcarmo/raspi-cluster
[cygwin]: http://www.cygwin.com
[uwsgi]: https://github.com/unbit/uwsgi
[wsl]: https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux
