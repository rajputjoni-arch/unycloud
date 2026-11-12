# Installation

UnyCloud is a single binary and can be used as standalone executable. It is a
maintained fork of File Browser and preserves the existing `filebrowser` CLI,
config, database, and `FB_*` environment behavior.

## Binary

Download `unycloud` from the
[UnyCloud releases page](https://github.com/trinity-labs/unycloud/releases) or
build it locally:

```sh
scripts/build.sh
```

Run it directly:

```sh
./dist/unycloud -r /path/to/your/files
```

Existing services that execute `/usr/local/bin/filebrowser` can keep doing so.
Install the UnyCloud artifact to that legacy path inside a mounted server root:

```sh
UNYCLOUD_INSTALL_ROOT=/mnt/server-root scripts/install-legacy-filebrowser.sh
```

UnyCloud is now up and running. Read the ["First Boot"](#first-boot) section for
more information.

## Docker

The Docker publishing workflow builds Linux/amd64 images from the existing
Dockerfile after all CI checks pass on pushed `v*` tags. Pull requests and
`master` pushes still run CI checks but do not publish images. Images are
published only to GitHub Container Registry (GHCR):

`ghcr.io/<github-owner>/unycloud`

The GHCR owner is taken from the repository running the workflow, so publishing
from a fork does not publish to the upstream owner's package. Version tags such
as `v0.29.3` produce `0.29.3` and `0.29` image tags; stable releases also receive
`latest`. Every build includes a `sha-<short-commit>` tag.

GHCR authentication uses the automatic `GITHUB_TOKEN` with `packages: write`;
no manually configured registry secrets are needed. Repository or organization
policies must allow the workflow to write packages. Set the GHCR package
visibility to public after the first publication if anonymous pulls are required.

### Publish a Version

Enable GitHub Actions in your fork and commit the workflow changes before
tagging. From the commit you want to release, create and push a new, unused
semantic version tag. Confirm that `origin` points to your fork:

```sh
git remote get-url origin
git tag -a v0.29.4 -m "Release v0.29.4"
git push origin v0.29.4
```

The tagged commit must contain the publishing workflow; merging to `master` is
not required to publish a tag. Follow the Continuous Integration run in your
fork's Actions tab. The existing GoReleaser job also creates a GitHub release
with binary archives for the tag.

### Use in aa-net-filebrowser

After the publishing job succeeds, replace the existing File Browser `FROM`
line in the downstream Dockerfile with the published version, substituting your
fork's GitHub owner:

```dockerfile
FROM ghcr.io/<github-owner>/unycloud:0.29.4
```

Keep the downstream customization steps and rebuild the downstream image:

```sh
docker build --pull -t aa-net-filebrowser:0.29.4 .
```

Use a specific version or image digest for reproducible deployments rather than
`latest`. Private images require registry authentication in the environment
building the downstream image. The published image uses BusyBox, runs as UID/GID
1000, and preserves `/bin/filebrowser`, `/srv`, `/config`, and `/database`.
Review downstream steps that need root, assume a package manager, or depend on
the old s6 image variant; this workflow does not publish an s6 variant. Test
customizations, volume permissions, authentication, and startup before deploying.

### Local Build

To build locally, build the binary first, then build the image from this working
tree:

```sh
scripts/build.sh
docker build -t unycloud:local .
```

The container keeps File Browser-compatible paths and still runs the executable
as `/bin/filebrowser` internally.

### Local BusyBox Image

```sh
docker run \
    -v filebrowser_data:/srv \
    -v filebrowser_database:/database \
    -v filebrowser_config:/config \
    -p 8080:80 \
    unycloud:local
```

Where `filebrowser_data`, `filebrowser_database` and `filebrowser_config` are
Docker [volumes](https://docs.docker.com/engine/storage/volumes/), where the
data, database and configuration will be stored, respectively. The default
configuration and database will be automatically initialized.

The default user that runs File Browser inside the container has UID 1000 and GID 1000. If, for one reason or another, you want to run the Docker container with a different user, please consult Docker's [user documentation](https://docs.docker.com/engine/containers/run/#user).

> [!NOTE]
>
> When using [bind mounts](https://docs.docker.com/engine/storage/bind-mounts/), that is, when you mount a path on the host in the container, you must manually ensure that they have the correct **permissions**. Docker does not do this automatically for you. The host directories must be readable and writable by the user running inside the container. You can use the [`chown`](https://linux.die.net/man/1/chown) command to change the owner of those paths.

UnyCloud is now up and running. Read the ["First Boot"](#first-boot) section for
more information.

## First Boot

Your instance is now up and running. UnyCloud will automatically bootstrap a
File Browser-compatible database, in which the configuration and the users are
stored. You can find the address in which your instance is running, as well as
the randomly generated password for the user `admin`, in the console logs.

> [!WARNING]
>
> The automatically generated password for the user `admin` is only displayed
> once. If you fail to remember it, you will need to manually delete the
> database and start UnyCloud again.

Although this is the fastest way to bootstrap an instance, we recommend you to take a look at other possible options, by checking [`config init`](cli/filebrowser-config-init.md) and [`config set`](cli/filebrowser-config-set.md), to make the installation as safe and customized as it can be.

If your goal is to have a public-facing deployment, we recommend taking a look at the [deployment](deployment.md) page for more information on how you can secure your installation.
