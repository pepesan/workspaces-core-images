![Logo][logo]
# Workspaces Core Images
This repository contains the base or **"Core"** images from which all other Workspaces images are derived.
These images are based off popular linux distributions and contain the wiring necessary to work within the Kasm platform.

While these images are primarily built to run inside the Kasm platform, they can also be executed manually.  Please note that certain functionality, such as audio, uploads, downloads, and microphone passthrough are only available within the Kasm platform.

```
sudo docker run --rm  -it --shm-size=512m -p 6901:6901 -e VNC_PW=password --build-arg START_XFCE4=1 kasmweb/<image>:<tag>
```

The container is now accessible via a browser : `https://<IP>:6901`

 - **User** : `kasm_user`
 - **Password**: `password`


For more information about building custom images please review the  [**How To Guide**](https://docs.kasm.com/docs/latest/how-to/workspaces-sessions/container-workspace/customization/building-images?utm_campaign=Github&utm_source=github)

The Kasm team publishes applications and desktop images for use inside the platform. More information, including source can be found in the [Default Images List](https://docs.kasm.com/docs/latest/how-to/workspaces-sessions/container-workspace/custom-images?utm_campaign=Github&utm_source=github)

# About Workspaces
Kasm Workspaces is a docker container streaming platform that enables you to deliver browser-based access to desktops, applications, and web services. Kasm uses a modern DevOps approach for programmatic delivery of services via Containerized Desktop Infrastructure (CDI) technology to create on-demand, disposable, docker containers that are accessible via web browser. The rendering of the graphical-based containers is powered by the open-source project   [**KasmVNC**](https://github.com/kasmtech/KasmVNC?utm_campaign=Github&utm_source=github)

![Screenshot][Kasm_Workflow]

Kasm Workspaces was developed to meet the most demanding secure collaboration requirements that is highly scalable, customizable, and easy to maintain.  Most importantly, Kasm provides a solution, rather than a service, so it is infinitely customizable to your unique requirements and includes a developer API so that it can be integrated with, rather than replace, your existing applications and workflows. Kasm can be deployed in the cloud (Public or Private), on-premise (Including Air-Gapped Networks), or in a hybrid configuration.

# Live Demo
A self-guided on-demand demo is available at [**kasm.com**](https://app.kasm.com/#/cast/kasmos?utm_campaign=Github&utm_source=github)

# Building Images
Build scripts for creating core images locally are in `scripts/` and require
[`yq`](https://github.com/mikefarah/yq) v4.53.3 (downloaded and sha256-verified automatically on first run; cached at `~/.cache/kasm/yq/`).

All commands must be run from the repository root.

Initialize git submodules first — these are required to build the KasmOS image:

```
git submodule update --init --recursive
```

## List available images

```
./scripts/build-image.sh --list-images
```

## Build an image

To build an image directly (tagged `local_build` by default):

```
./scripts/build-image.sh --build kasmweb/core-ubuntu-jammy
```

To inspect the `docker build` command before running it:

```
./scripts/build-image.sh --list-image-build-command kasmweb/core-ubuntu-jammy
```

To list build commands for all images:

```
./scripts/build-image.sh --list-images-build-commands
```

Override the image tag with `-t`:

```
./scripts/build-image.sh --build kasmweb/core-ubuntu-jammy -t my-tag
```

# Ubuntu 26.04 (Resolute) Support

This fork adds an experimental `kasmweb/core-ubuntu-resolute` image, built on top of
`ubuntu:26.04` ("Resolute Raccoon"), using the same generic `dockerfile-kasm-core`
pipeline as `core-ubuntu-jammy` and `core-ubuntu-noble`. It has been built and
smoke-tested locally (XFCE session starts, KasmVNC authentication works, the web
UI responds on `:6901`) and pushed to Docker Hub as `pepesan/core-ubuntu-resolute`.

## Known limitation: no Kasm Profile Sync binary for this distro

The upstream Kasm team has not yet published a `kasm-profile-sync` /
`kasm-profile-sync-2` binary build for `ubuntu_resolute` in their build-artifacts
S3 bucket (`kasmweb-build-artifacts.s3.amazonaws.com`). Requesting that binary
returns `403 Forbidden` — this was verified to also be the case for every other
Ubuntu codename newer than `noble` (`oracular`, `plucky`, `questing`), so it is not
specific to Resolute: Kasm simply has not built profile-sync for any post-Noble
Ubuntu release yet.

Because of this, the `RUN bash $INST_SCRIPTS/profile_sync/install_profile_sync.sh`
step in `dockerfile-kasm-core` was changed to `... || true`, so a missing binary no
longer aborts the build for any distro. This has no effect on `jammy`/`noble` (the
binary still downloads and installs normally there); it only changes behavior for
distros where the binary is unavailable.

**Practical impact:** containers built from `core-ubuntu-resolute` do not have
`/usr/bin/kasm-profile-sync` / `-2`. The startup and shutdown hook scripts already
check for the binary's presence and skip profile persistence gracefully when it is
missing (`"Profile sync not available"`), so the container still starts and runs
fine. The only feature that does not work is **persisting the user's home
directory/profile across ephemeral Kasm sessions** via the Kasm Workspaces
platform's object storage. This only matters when running inside the full Kasm
Workspaces platform with `KASM_PROFILE_LDR` set; a plain `docker run` (as used for
local testing) never exercises this code path regardless.

## Other fixes needed for Ubuntu 26.04

- `src/ubuntu/install/fonts/install_custom_fonts.sh`: the Ubuntu language-pack
  install used to run as a single `apt-get install` with the full `LOCALES_UBUNTU`
  list. Two packages, `language-pack-ga` (Irish) and `language-pack-ia`
  (Interlingua), no longer exist in the `resolute` archive, which made the whole
  command fail. Packages are now installed one at a time, so a single missing
  language pack no longer aborts the build (no behavior change for `jammy`/`noble`,
  where every package in the list is still available).
- `src/ubuntu/install/squid/install/install_squid.sh`: `chown -R proxy:proxy
  /usr/local/squid -R` passed `-R` twice. Ubuntu 26.04 ships the Rust-based
  `uutils-coreutils` as `/usr/bin/chown` (replacing GNU coreutils), which rejects
  duplicate flags instead of silently tolerating them like GNU `chown` did. Fixed
  by removing the duplicate flag.

[logo]: https://5856039.fs1.hubspotusercontent-na1.net/hubfs/5856039/Kasm_Workspaces_Logo.png "Kasm Logo"
[Kasm_Workflow]: https://5856039.fs1.hubspotusercontent-na1.net/hubfs/5856039/dockerhub/launching_ubuntu_jammy.gif "Kasm Workflow"

# Reporting Issues

To report any issues for this repository, please use our central issue tracker: **[Kasm Workspaces Issue Tracker](https://github.com/kasmtech/workspaces-issues/issues)**
