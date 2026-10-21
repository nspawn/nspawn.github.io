---
layout: post
title: 'Nspawn 2.0: support for compose v2 and Dockerfile builds'
date: 2026-10-21 02:42 -0500
categories: [linux, systemd, nspawn, containers, docker, compose, dockerfile, oci]
tags: [linux, systemd, nspawn, containers, docker, compose, dockerfile, oci]
author: edu4rdshl
image:
  path: /nspawn-2-0-compose.png
  alt: The docker compose octopus next to the nspawn logo
excerpt: nspawn 2.0.0 runs docker compose files, builds Dockerfiles with its own builder, gives proxies and dashboards a read-only Docker API and hands GPUs to machines through CDI.
---

## Since 1.0.0

nspawn 1.0.0 came out a month ago. There were ten minor releases between that one and 2.0.0, and most of them went into making a machine do what `docker run` does with a container, so that compose files could run on top:

- 1.1.0: restart policies, resource limits, `cp` and `--json` output.
- 1.2.0: `run` stays attached like `docker run`, networks of your own, `events`, `stats`, `kill` and `update`.
- 1.3.0: several networks per machine with names on each, healthchecks, secrets and the rest of `docker run`'s flags.
- 1.4.0: `--network container:NAME` and parallel layer downloads.
- 1.5.0: signature verification. Every image on the hub is signed with cosign by the workflow that builds it, and a pull verifies a signature before any layer comes down. That was the first item of the "What is missing" list in the last post.
- 1.6.0: `--interface IFACE` moves a host interface into the machine, a wifi adapter included. That's what the [post about wireless adapters](https://blog.nspawn.org/posts/wireless-adapters-on-systemd-nspawn-containers/) was about.
- 1.8.0: the output of the machines goes to a journald namespace of nspawn's own, with `--log-driver`.
- 1.9.0: TAB completes the names of machines, images, networks and volumes.
- 1.10.0: `--depends-on`, compose's `depends_on` for a single machine.

2.0.0 is the release all of that was for.

## nspawn compose

`nspawn compose` takes the files and commands of docker compose 5.6.0, with its flags, output and errors:

```bash
$ nspawn compose config
$ nspawn compose up -d
$ nspawn compose ps
$ nspawn compose logs -f web
$ nspawn compose down
```

The files load the way docker compose loads them: interpolation, `.env`, profiles, `include`, `extends`, override files, `!reset` and `!override`, and the schema checks with the same messages. `nspawn compose config` prints what `docker compose config` prints.

Each replica of a service is a machine called `PROJECT-SERVICE-N`, with docker compose's labels. Networks, volumes, secrets and configs are nspawn's, and a machine is made again when its config hash or its image changes. `depends_on` is handled by compose itself: it orders the starts and waits for the conditions, and the machines carry no `--depends-on`, so a service others depend on can be recreated, scaled down or removed while they run.

`alpha`, `attach`, `bridge`, `commit`, `export`, `publish`, `push` and `watch` are refused with a reason. [COMPOSE.md](https://github.com/nspawn/nspawn/blob/2.0.0/docs/COMPOSE.md) lists every attribute and option and what nspawn does with it: supported, partly supported, ignored with a warning, or refused. A service that uses a refused attribute fails to start with the reason.

There are plans to extend this further, but for now we focused on supporting the most used options.

## Dockerfile builds

`nspawn build` used to support mkosi builds only. It now builds Dockerfiles too, with a builder of its own. A directory with a `Dockerfile` (or `Containerfile`) is a Dockerfile build, and one with `mkosi.conf` and no Dockerfile is still an mkosi build:

```bash
$ nspawn build -t team/app:1 .
$ nspawn build -f Dockerfile.prod --target runtime --build-arg VERSION=2 -t team/app:2 .
```

The Dockerfile is read the way BuildKit v0.33.1's frontend reads it. Multi-stage builds, `ARG`, `ONBUILD`, heredocs, `COPY --from --chown --chmod --parents --exclude`, `ADD` of archives and of URLs with `--checksum`, and `RUN --mount` of the cache, bind, tmpfs and secret kinds all work, and so do `--secret`, `--network`, `--add-host`, `--label`, `--pull`, `--no-cache`, `--no-cache-filter`, `-q`, `--iidfile` and `--push`. `.dockerignore` and `Dockerfile.dockerignore` are honored.

Every `RUN` runs under `systemd-nspawn` with the same confinement as an app machine. A layer is the same tar BuildKit writes for that tree, and a build that comes entirely from the cache gives the same image ID.

The cache is listed with `nspawn builder du` and pruned with `nspawn builder prune` (`-a`, `--filter until=`, `--keep-storage`). Compose's `build:` sections, `compose build` and `compose up --build` use the same builder.

## A Docker API for proxies and dashboards

Traefik, caddy-docker-proxy and the like find their backends by reading docker's socket. A machine that mounts `/var/run/docker.sock` or `/run/docker.sock` from the host, or sets compose's `use_api_socket`, now gets nspawn's Docker API on that path: API 1.24 to 1.51, with the machines as containers (labels, networks and addresses), inspect, events, logs, stats, top and the networks. It is read-only, except for `POST /containers/ID/kill`.

This is Traefik's quick start, and Traefik finds `whoami` through the socket as it would on docker:

```yaml
services:
  reverse-proxy:
    image: traefik:v3
    command: --api.insecure=true --providers.docker
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"
```

caddy-docker-proxy, nginx-proxy with docker-gen and acme-companion, homepage and Dozzle work too. Each machine gets its own socket, `nspawn-api@NAME.socket`, opened before the machine starts.

## Other additions

- **auto-update:** `nspawn auto-update` updates the machines labelled `org.nspawn.auto-update=registry` (podman's `io.containers.autoupdate=registry` works too): the machine is made again from the image its tag points to now, with every option it had, and goes back to the old image if the new one doesn't come up. `--dry-run` lists the pending updates. The packages ship `nspawn-auto-update.timer`, daily and disabled.
- **s6-overlay images:** Images whose `/init` supervises the rest, like linuxserver.io's and Home Assistant's, run with `/init` as PID 1. `--init=false` does the same for any app, `--init` puts the program under nspawn's stub init, and compose's `init:` maps to them.
- **GPUs and CDI:** `--device VENDOR/CLASS=NAME` takes the devices of the CDI specs in `/etc/cdi` and `/var/run/cdi` (nodes, libraries, variables and hooks), and `--gpus all|N|device=...` the NVIDIA or AMD ones. Compose's `gpus:`, device reservations and CDI names in `devices:` work too.
- **docker's scripting flags:** `--env-file`, `--label-file`, `--mount`, `--ip`, `ps -q`, `ps --filter`, `ps --format` with Go templates, `inspect -f`, `images prune`, `system df`, `system prune`, `network connect` and `disconnect` on running machines, `wait` and `port`:

  ```bash
  $ nspawn ps --filter label=com.docker.compose.project=blog --format '{{.Names}}\t{{.Status}}'
  ```

- An app's network namespace allows unprivileged ping and ports below 1024, and `--sysctl` can override both.
- `-u 1000:1000` works on distroless and scratch images.

## What changed

Some of these can break existing setups:

- `build -f` names the Dockerfile, as with docker. `--force`, which replaces an image of the same name, has no short form any more.
- `exec` runs as the machine's user (its `--user` or the image's `USER`), like `docker exec`. `-u root` gives root. On booted machines it's still root.
- `kill` on an app without a stop signal of its own ends its restarts, whatever the signal.
- A `--depends-on NAME:completed` dependency runs until one run of it succeeds, not at every start of the machines that depend on it.
- An image with an init is booted only when it names no program, or the program is `init`, `/sbin/init`, `/usr/sbin/init` or systemd. `/init` and anything else run as an app. Installed images keep their mode until they are pulled again.
- `run` and `create` look for a local image first, under the configured registry and then as Docker Hub's, and pull only when neither is there.
- `machines_dir` has to be one of the directories systemd-nspawn finds machines in, and `state_dir` can't be inside a directory machined lists images from or hold one, because `machinectl clean` or `remove` would delete nspawn's data.
- polkit asks for a password only when the call carries `ALLOW_INTERACTIVE_AUTHORIZATION`, as nspawn, busctl and systemctl set it. A call without it that needs one is refused at once.
- `daemon --install` writes new units: `nspawn-api@.socket`, `nspawn-api@.service`, the auto-update service and timer, and journald's config for the `nspawn` namespace. If you installed the binary by hand, run it again after upgrading.

The [release notes](https://github.com/nspawn/nspawn/blob/2.0.0/docs/releases/2.0.0.md) have the full list, including the rest of the docker semantics (`--entrypoint`, `--tmpfs`, stop timeouts, `logs`) and the changes on D-Bus.

## Known gaps

- mstack is still experimental and untested in this release. No host of the test matrix runs it, since it needs systemd 261 or a release after 262 (262 can't boot a managed user namespace, [systemd/systemd#43899](https://github.com/systemd/systemd/issues/43899)). The Docker API, CDI devices and `network connect` are not available on mstack machines.
- GPUs were tested only with CDI specs written for the test suite, not with real NVIDIA or AMD hardware.
- The builder doesn't do `--ssh`, outputs other than a local image (`--output`, `--cache-to`), named or remote contexts, `ADD` of a git repository, attestations, other platforms, `RUN --device`, `RUN --security=insecure` or `--check` and `--call`. `--cache-from` is ignored and `COPY --link` is a plain `COPY`.

The requirements are the same as for 1.10.0, and the packages for Fedora, Debian/Ubuntu and Arch are attached to the [release](https://github.com/nspawn/nspawn/releases/tag/2.0.0). If a compose file or a Dockerfile that works on docker fails on nspawn, open an issue with the file.

## Closing words

We have a lot ideas in mind to implement in the future, but the stability, reliability and security of this software is our priority. Each release adds new code that needs to be tested every time that new features are added so that there are not regressions, bugs, etc (or at least as little as possible), so the cadence of the releases might be slower. Even with the use of AI, for the 2.0.0 release we had ~4 days of pure testing between the e2e, real-world scenarios and fixes before releasing it.

We hope that you enjoy this release. As usual, bug reports/feature requests are welcome.
