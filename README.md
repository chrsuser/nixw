# nixw — a Gradle-wrapper-style launcher for Nix flakes

`nixw` is a single POSIX `sh` script that you commit to a flake-based project.
Anyone can then build it with `./nixw build`, `./nixw develop`, `./nixw run`, …
even if they have never installed Nix.

```console
$ ./nixw build
nixw: downloading nix-user-chroot 2.1.1 (x86_64)
nixw: installing Nix 2.35.4 into /home/alice/.nix (this only happens once)
nixw: running the Nix installer (this can take a minute)...
nixw: Nix 2.35.4 installed
$ ./nixw develop
```

`./nixw ARGS` is the same as `nix ARGS`:

- **Nix is already installed** (on `PATH`, or in a standard location such as
  `/nix/var/nix/profiles/default`): nixw checks that the `nix-command` and
  `flakes` features are enabled, then runs that `nix` unchanged. If either
  feature is off, nixw prints how to enable it and exits.
- **Nix is not installed** (Linux only): nixw downloads a pinned static
  [nix-user-chroot](https://github.com/nix-community/nix-user-chroot) binary and
  the pinned official Nix installer, verifying both against SHA-256 checksums
  stored in the script. It then runs
  `install --no-daemon --no-channel-add --no-modify-profile` inside the chroot,
  which installs into `~/.nix`. Later runs execute that Nix directly inside the
  chroot. No root access, no daemon, and no changes to your shell profile.

## Adding nixw to a project

```sh
curl -fsSLO https://<where-you-host-it>/nixw && chmod +x nixw
git add nixw    # git keeps the executable bit
```

Requirements on the user's machine, for the first run only: `curl` or `wget`,
`tar`, `xz`, and one of `sha256sum`/`shasum`/`openssl`. The kernel must also
allow unprivileged user namespaces (see [Troubleshooting](#troubleshooting)).

## How the private install is laid out

Everything lives under `~/.nix`, which is mounted at `/nix` inside the chroot:

| Path (in `~/.nix`)                     | What it is                                          |
|----------------------------------------|-----------------------------------------------------|
| `store/`, `var/nix/`                   | an ordinary single-user Nix store                   |
| `etc/nix/nix.conf`                     | private config, enabling `nix-command flakes` (nix-user-chroot sets `NIX_CONF_DIR=/nix/etc/nix`) |
| `var/nix/profiles/nixw/nix-<version>`  | one profile per pinned Nix version (a GC root)      |
| `var/nixw/nix-user-chroot-<version>`   | the downloaded chroot binary                        |
| `var/nixw/nix-<version>.installed`     | marker written after a successful install           |

The installer runs with a scratch `HOME` whose `.nix-profile` points at the
per-version profile. Your real home therefore gets no `~/.nix-profile`,
`~/.nix-channels` or shell-profile edits. It also means projects pinned to
different Nix versions can share one `~/.nix`, and each version is installed
once, just as Gradle keeps several distributions side by side.

When several nixw runs start at once, a lock directory makes them install only
once. An interrupted install is retried on the next run.

## Configuration

Project maintainers set the pins by editing the variables at the top of the
script:

```sh
nixw_nix_version=2.35.4
nixw_nix_installer_sha256=…   # sha256 of https://releases.nixos.org/nix/nix-<version>/install
nixw_chroot_version=2.1.1
nixw_chroot_sha256_x86_64=…   # sha256 of each nix-user-chroot-bin-<version>-<arch>-unknown-linux-musl
nixw_chroot_sha256_aarch64=…
nixw_chroot_sha256_i686=…
```

The versioned Nix installer embeds and checks the hash of the Nix tarball, so
pinning the installer's hash pins the whole download. To bump Nix:

```sh
curl -fsSL https://releases.nixos.org/nix/nix-X.Y.Z/install | sha256sum
```

Users can override settings with environment variables:

| Variable                     | Effect                                                          |
|------------------------------|-----------------------------------------------------------------|
| `NIXW_NIX_DIR`               | where the private Nix lives (default `~/.nix`)                  |
| `NIXW_USE_SYSTEM_NIX=0`      | ignore any installed Nix and use the private one                |
| `NIXW_SKIP_CHECK=1`          | skip the feature check on an installed Nix                      |
| `NIXW_VERBOSE=1`             | print which Nix is used and what is downloaded                  |
| `NIXW_NIX_VERSION`           | use a different Nix version (with `NIXW_NIX_INSTALLER_SHA256`; without it, the installer isn't verified and nixw warns) |
| `NIXW_NIX_INSTALLER_URL`     | fetch the installer from a mirror                               |
| `NIXW_CHROOT_URL`, `NIXW_CHROOT_SHA256` | fetch nix-user-chroot from elsewhere                 |

## Limitations

- **Store paths only exist inside the chroot.** With the private Nix, `./result`
  links to `/nix/store/…`, which doesn't exist on the host. Use `./nixw run`, or
  run host commands inside the chroot with `nix shell`. Your host tools stay
  visible there:

  ```sh
  ./nixw shell . --command cp -rL result dist     # copy build output out
  ./nixw develop --command make                   # work in the dev shell
  ```

  Running `./nixw` again from inside `./nixw develop` or `./nixw shell` works:
  it finds the private `nix` on `PATH`.
- **The private install is Linux-only** (x86_64, aarch64, i686), because it
  depends on Linux user namespaces. On macOS and other systems nixw only works
  with an installed Nix, and otherwise points to the Nix installer.
- **An installed Nix is used as-is.** nixw doesn't enforce the pinned version
  on a Nix you installed yourself.

## Troubleshooting

When nix-user-chroot fails its first-run check, nixw prints the error and
explains the cause:

- **`kernel.unprivileged_userns_clone = 0`** (older Debian) or
  **`user.max_user_namespaces = 0`**: user namespaces are disabled, and an
  admin has to enable them with `sysctl`.
- **Ubuntu 23.10 and later** (`kernel.apparmor_restrict_unprivileged_userns = 1`):
  an admin has to add an AppArmor profile that grants the chroot binary
  `userns`. nixw prints the exact profile to use.
- **Inside Docker and similar containers**, seccomp usually blocks `unshare`.
  Installing Nix normally in the image is simpler.

The installer's output is written to `~/.nix/var/nixw/install.log` and shown
only if the install fails.

## Uninstalling

Store files are read-only, so make them writable first:

```sh
chmod -R u+w ~/.nix && rm -rf ~/.nix
```

Nix's usual cache in `~/.cache/nix` can be removed too.

## License

MIT. See [LICENSE](LICENSE). The script carries an SPDX header, so the notice
stays with copies committed to other projects.
