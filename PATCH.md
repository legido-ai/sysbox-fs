# Patch branch: master (contains the seccomp POLLERR fix)

This branch exists because upstream's fix for `nestybox/sysbox#1046` has **never been
released**, and we run it in production.

| | |
|---|---|
| Base | **`master`** @ `201ec13` — 20 commits ahead of the 0.7.1 base |
| Contains | `1ac3fd58` *"Keep the seccomp session open on a transient POLLERR"* (2026-09-29), issue closed 2026-10-18 |
| Delta from upstream master | **this file only** |
| Upstream release status | `sysbox` v0.7.1 (2026-07-31) is still the latest; `sysbox-fs` has no tags |

## Why master and not 0.7.1 + one commit

An earlier revision of this branch cherry-picked only `1ac3fd5` onto `c3d2ebc6` (the
0.7.1 release commit) to keep the delta to one commit. That was withdrawn: it creates
a fork we would have to maintain indefinitely, and it omits fixes we want.

What master adds over 0.7.1 (20 commits) includes:

| Commit | Fix |
|---|---|
| `1ac3fd58` | **the seccomp POLLERR fix** — the reason this branch exists |
| `b2652851` | seccomp: immutable-remount bypass via spoofed fstype (**security**) |
| `f985b753` | seccomp: incorrect procfs/sysfs base-mount detection |
| `3996858` | mount: honour kernel precedence for mount requests |
| `dfcd670` | seccomp: execute authorized remounts instead of resuming via CONTINUE |
| `48985ee6` | state: don't hold the container-state lock across fuse mount/unmount |
| `00842aae` | `/proc/uptime` seekable, so `procps` can refresh |
| `a2d5dcfd` | FUSE requests with unresolved (pid=0) requester |
| `6b86db4`, `dd2813b`, `3e1e94f`, `700f165` | nsenter personality and capability handling |
| `016d075`, `898265b`, `3932280`, `2e12024` | dependency bumps, one reverted, `go mod tidy` |

Several are security-relevant, so running master is not merely "the fix plus
surprises" — it is the maintained branch.

## IPC compatibility with 0.7.1 `sysbox-mgr`

Checked before building, because a mismatched gRPC contract with the 0.7.1
`sysbox-mgr` on the host would be silent and ugly:

- `sysbox-fs`'s `go.mod` uses `replace github.com/nestybox/sysbox-ipc => ../sysbox-ipc`
  (build-time sibling, not a published version).
- `sysboxFsProtobuf.proto` is **byte-identical** between the 0.7.1-era pin `f05151f4`
  and `sysbox-ipc` master (sha256 `30799f9302077640`, 1614 bytes).

So a master `sysbox-fs` can run against a 0.7.1 `sysbox-mgr`/`sysbox-runc`.

## Build

```sh
# Go 1.25.x (sysbox-fs go directive: 1.25.0), protoc, and the two protoc plugins
mkdir -p /src && cd /src
git clone https://github.com/legido-ai/sysbox-fs.git && git -C sysbox-fs checkout legido/master
git clone https://github.com/legido-ai/sysbox-ipc.git && git -C sysbox-ipc checkout f05151f4b4c1df63d7fd241577ca032905c1bd0e

export PATH=$PATH:/root/go/bin
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.36.5
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.5.1

make -C sysbox-ipc sysbox-ipc                    # generates the .pb.go stubs
make -C sysbox-fs TARGET_ARCH=arm64 VERSION=0.7.1 \
     EDITION="Community Edition (CE)" sysbox-fs
```

## Install

```sh
sudo install -m0755 /usr/bin/sysbox-fs /usr/bin/sysbox-fs.pre-1ac3fd5   # keep stock first
sudo install -m0755 ./sysbox-fs /usr/bin/sysbox-fs
# drain the box (docker stop sysbox) before restarting the services
sudo systemctl restart sysbox-mgr sysbox-fs
```

## Verify

```sh
/usr/bin/sysbox-fs --version                    # must not report c3d2ebc6
docker exec sysbox docker run --rm busybox true
docker exec sysbox journalctl -u docker | grep -i "function not implemented"   # empty
```

## Delete this fork when

A `sysbox` release contains `1ac3fd58`:

```sh
curl -s https://api.github.com/repos/nestybox/sysbox/releases/latest | grep tag_name
```

Then upgrade normally and drop the local build. `sysbox-fs` has never been tagged —
watch the **parent** repo's releases.
