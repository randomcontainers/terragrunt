# terragrunt

Container images with [Terragrunt](https://terragrunt.com/), which runs OpenTofu across many units of infrastructure code. A unit is a directory with a `terragrunt.hcl` file. Terragrunt fills in settings that units share through included files, passes outputs from one unit to another through `dependency` blocks, and runs `tofu` in each unit in dependency order. `terragrunt` is compiled from the release tag with a current Go release on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`. The images are rebuilt when Terragrunt publishes a release, when the pinned Go release is raised and when the base image changes. The default image adds OpenTofu, which Terragrunt runs for every unit, Git for modules from the OpenTofu registry and from Git repositories, and the OpenSSH client for Git over SSH.

This is an unofficial build, not affiliated with or endorsed by Gruntwork, Inc., which develops Terragrunt. Report problems with the image in this repository and problems with Terragrunt itself [upstream](https://github.com/gruntwork-io/terragrunt/issues).

## Quick start

Save this as `main.tf`:

```hcl
terraform {
  required_providers {
    random = {
      source = "hashicorp/random"
    }
  }
}

variable "prefix" {
  type = string
}

resource "random_pet" "name" {
  prefix = var.prefix
}

output "name" {
  value = random_pet.name.id
}
```

and this as `terragrunt.hcl` next to it:

```hcl
inputs = {
  prefix = "dev"
}
```

Look at the plan, then apply it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt run -- plan
docker run --rm -it --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt run -- apply
```

Terragrunt copies the unit into `.terragrunt-cache/`, runs `tofu init` there when the unit needs it, and passes `inputs` to OpenTofu as `TF_VAR_` variables. `tofu apply` asks for confirmation, hence `-it`; in CI pass `-auto-approve` after `apply`. This unit has no backend, so `apply` writes the state into `.terragrunt-cache/` as well (see [Files and permissions](#files-and-permissions)). A few more commands:

```sh
# Plan every unit below the current directory, in dependency order
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt \
  run --all -- plan

# List the units below the current directory in the order they run
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt find --dag

# Check the formatting of every HCL file below the current directory
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt hcl fmt --check

# Generate the units of the terragrunt.stack.hcl file in the current directory
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/terragrunt stack generate
```

Without arguments the image prints `terragrunt --help`. In CI, pin a major version tag such as `1` or a digest. A minor version tag such as `1.2` stops getting updates once the next minor release is built (see [Tags](#tags)).

## Credentials and settings

Terragrunt, OpenTofu and the providers take credentials from environment variables, and most Terragrunt flags can be set with a `TG_` variable too, such as `TG_NON_INTERACTIVE=true` for `--non-interactive`. `-e NAME` without a value passes the variable from your shell:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e AWS_SESSION_TOKEN -e AWS_REGION \
  ghcr.io/randomcontainers/terragrunt run --all -- plan
```

In the container `HOME` is not your home directory (it is `/` for a UID without a user in the image), so files such as `~/.aws/credentials` and `~/.tofurc` are not there. Mount what you need read-only and point to it:

- `-v "$HOME/.aws:/aws:ro" -e AWS_SHARED_CREDENTIALS_FILE=/aws/credentials -e AWS_CONFIG_FILE=/aws/config` for the AWS provider and an S3 backend in `remote_state`.
- `-e TF_CLI_CONFIG_FILE=/work/.tofurc` for an OpenTofu CLI configuration file in your directory.
- `-e TF_TOKEN_app_terraform_io`, one variable per host, for the API tokens of private registries and remote backends.

## Providers and the provider cache

`tofu init` downloads the providers of each unit into that unit's `.terragrunt-cache/`, so later commands in new containers find them there. The images contain no providers.

With OpenTofu 1.10 or later, Terragrunt normally shares one provider cache between all units, in `$XDG_CACHE_HOME/terragrunt/providers` (`/cache/terragrunt/providers` here), and OpenTofu links each unit's providers to it. `/cache` starts empty in every new container, where those links would lead nowhere, so the images set `TG_NO_AUTO_PROVIDER_CACHE_DIR=true` to turn the shared cache off. To use it, keep `/cache` in a volume and turn it back on:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -v terragrunt-cache:/cache -e TG_NO_AUTO_PROVIDER_CACHE_DIR=false \
  ghcr.io/randomcontainers/terragrunt run --all -- plan
```

Mount the same volume at `/cache` for every later command in that directory. The volume also keeps the clones of Git sources that Terragrunt stores in `/cache/terragrunt/cas`.

`terragrunt exec` does not read `TG_NO_AUTO_PROVIDER_CACHE_DIR`, so it still points OpenTofu at the shared cache. A `tofu init` that it runs, as in `terragrunt exec -- tofu init` or with `--in-download-dir`, links the providers into `/cache`. Run `tofu init` through `terragrunt run`, or keep `/cache` in a volume when you use `exec`.

Providers are separate programs that `tofu` starts. Most providers in the registry are static Go binaries, which run on the Alpine images too; a provider that needs glibc runs only on the Ubuntu images.

## Modules and Git

Terragrunt downloads the module of a unit from the `source` in its `terraform` block, and OpenTofu downloads the modules that the module calls. Both run `git` for Git sources, and modules from the OpenTofu registry are Git repositories too. The default image has Git and the OpenSSH client, so sources such as `git::https://example.com/infra/modules.git//network?ref=v1.2.0` and `git::ssh://git@github.com/example/modules.git//network?ref=v1.2.0` work. For SSH, pass your ssh-agent and a `known_hosts` file:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -v "$SSH_AUTH_SOCK:/agent.sock" -e SSH_AUTH_SOCK=/agent.sock \
  -v "$HOME/.ssh/known_hosts:/ssh/known_hosts:ro" \
  -e GIT_SSH_COMMAND="ssh -o UserKnownHostsFile=/ssh/known_hosts" \
  ghcr.io/randomcontainers/terragrunt run --all -- plan
```

On Docker Desktop, mount `/run/host-services/ssh-auth.sock` instead of `$SSH_AUTH_SOCK`.

`ssh` works for any UID given with `--user`; [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client) explains how.

## What is in the image

| | slim | default |
|---|---|---|
| `terragrunt` in `/usr/local/bin`, built without cgo, so it is statically linked and loads no libraries | yes | yes |
| OpenTofu (`tofu`), which Terragrunt runs for every unit | no | yes |
| Git, for modules from the registry and `git::` sources | no | yes |
| The OpenSSH client (`ssh`, `scp`, `ssh-keygen` and the other client tools), for Git over SSH | no | yes |

Not included: providers, Terraform, cloud command-line tools such as the AWS CLI or `kubectl`, and linters such as TFLint. OpenTofu is the `slim` build of [randomcontainers/opentofu](https://github.com/randomcontainers/opentofu), Git is the distro's `git` package, and the OpenSSH client is the [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client) build.

The Go version, the commit and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/terragrunt/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work: Terragrunt runs `tofu` for every unit, modules from the registry and from Git repositories need Git, and Git over SSH needs `ssh`. `slim` has `terragrunt` alone. It formats, renders and lists configurations and generates stacks whose units come from local paths. `terragrunt run` needs `tofu`, and sources from Git or the registry need `git`. Use it as the base for your own image.

The default image includes Git, which is licensed under the GNU General Public License (GPL-2.0-only). Use `slim` if your policy excludes GPL software. The default image is also published as `ghcr.io/randomcontainers/terragrunt-opentofu-openssh-client`, built in the [terragrunt-opentofu-openssh-client](https://github.com/randomcontainers/terragrunt-opentofu-openssh-client) repository with the same contents and a different digest.

## Tags

`<version>` is a Terragrunt release such as `1.2.0`. `<minor>` and `<major>` are its shorter forms, `1.2` and `1`. They follow the newest release in their series while that series is the newest one. Once 1.3.0 is built, `1.2` stays on the last 1.2 image and is no longer rebuilt, since only the newest release is built (see [Updates](#updates)). The default image carries the OpenTofu and OpenSSH client releases that were current when it was built, and the tags do not name them.

| Default (with OpenTofu, Git and the OpenSSH client) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Terragrunt version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and `/cache` can be written by anyone. Terragrunt keeps its caches under `/cache/terragrunt` (see [Providers and the provider cache](#providers-and-the-provider-cache)). How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Terragrunt writes a `.terragrunt-cache/` directory into each unit it runs, which holds the copy of the unit that `tofu` runs in, with its `.terraform/` directory. After `tofu init` it copies `.terraform.lock.hcl` next to `terragrunt.hcl`, where you commit it. `stack generate` writes the units of a stack to `.terragrunt-stack/` next to the `terragrunt.stack.hcl` file. Keep `.terragrunt-cache/` and `.terragrunt-stack/` out of version control.

Without a `remote_state` block in `terragrunt.hcl` or a `backend` block in the module, OpenTofu writes `terraform.tfstate` into the unit's copy in `.terragrunt-cache/`. Deleting that directory deletes the state. The name of the copy's directory is derived from the unit's path in the container, so running the unit from another mount, for example with its parent directory mounted at `/work`, starts from an empty state. Use a remote backend for infrastructure you keep. A state file holds every attribute of your resources in plain text, secrets included, unless you configure [state encryption](https://opentofu.org/docs/language/state/encryption/).

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Terragrunt release and are rebuilt when the base image changes. To run units you need `tofu` as well. This copies it, with its license files and sources, from the OpenTofu image of the same distro and adds Git:

```dockerfile
FROM ghcr.io/randomcontainers/opentofu:slim-ubuntu@sha256:... AS opentofu

FROM ghcr.io/randomcontainers/terragrunt:slim-ubuntu@sha256:...
COPY --from=opentofu /usr/local/ /usr/local/
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from the `slim-alpine` tags and use `apk add --no-cache git`. The entrypoint is `["tini", "--", "terragrunt"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new releases and base image fixes, let Dependabot or Renovate update the digests in your `FROM` lines.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/terragrunt/` holds the version, the source links, the build details, the license files, the MPL-2.0 module sources and `runtime-deps`, the list of distro packages `terragrunt` needs at run time beyond those in the base image (none).

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/terragrunt:latest \
  --repo randomcontainers/terragrunt --signer-repo randomcontainers/ci
```

Images published since November 2026 also have a keyless cosign signature from the same workflow, for admission controllers that check cosign signatures. Tags last built before then have only the attestation.

```sh
cosign verify ghcr.io/randomcontainers/terragrunt:latest \
  --certificate-identity https://github.com/randomcontainers/ci/.github/workflows/build.yml@refs/tags/v1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository randomcontainers/terragrunt
```

Images from `ghcr.io/randomcontainers/terragrunt-opentofu-openssh-client` are built in [randomcontainers/terragrunt-opentofu-openssh-client](https://github.com/randomcontainers/terragrunt-opentofu-openssh-client), so use `randomcontainers/terragrunt-opentofu-openssh-client` in `--repo` and `--certificate-github-workflow-repository`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/terragrunt:latest --format '{{ json .SBOM }}'
```

Terragrunt publishes prebuilt binaries but no source tarball, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [gruntwork-io/terragrunt](https://github.com/gruntwork-io/terragrunt) and stops unless the tag points to the commit recorded in `package.yml`. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile, and the Go modules come from the Go module proxy and are checked against Terragrunt's `go.sum`. The binary records the commit it was built from, and `buildinfo` lists it with the Go modules.

## Updates

The project checks the releases of [gruntwork-io/terragrunt](https://github.com/gruntwork-io/terragrunt/releases) every 15 minutes. A release is picked up once it is 24 hours old; release candidates and other pre-releases are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Go toolchain is pinned in the Dockerfiles and raised by hand when Go publishes a security release, and each raise rebuilds the images of the current version. Terragrunt sometimes raises the Go version its `go.mod` requires, as 1.2.0 did, and such a release does not build until the pin is at least that version.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new OpenTofu or OpenSSH client image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t terragrunt:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel. The build downloads several hundred megabytes of Go modules. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

Terragrunt is released under the [MIT License](https://github.com/gruntwork-io/terragrunt/blob/main/LICENSE.txt). Its `internal/vendored/opentofu` directory holds the code of OpenTofu's built-in functions under the Mozilla Public License 2.0, with code from Go's `net` package (BSD-3-Clause) in its `ipaddr` package, and `internal/vendored/compress/flate` holds code from Go (BSD-3-Clause). The binary also contains:

- the Go standard library (BSD-3-Clause);
- the Go modules Terragrunt depends on, under MIT, Apache-2.0, BSD-3-Clause, BSD-2-Clause and MPL-2.0, among them HashiCorp's go-getter and HCL, SOPS, and Gruntwork's boilerplate and terragrunt-engine-go under MPL-2.0;
- the Unicode data in go-textseg versions 15 (Unicode-DFS-2016) and 17 (Unicode-3.0);
- the Liberation Mono font (OFL-1.1), which chroma embeds for its SVG output.

The image's license label is `MIT AND MPL-2.0 AND Apache-2.0 AND BSD-3-Clause AND BSD-2-Clause AND Unicode-DFS-2016 AND Unicode-3.0 AND OFL-1.1`. The license files are in `/usr/local/share/randomcontainers/terragrunt/licenses/`, with one directory per Go module. The directory of `github.com/opencontainers/go-digest` also has its `LICENSE.docs` (CC-BY-SA-4.0), and the license of `github.com/modelcontextprotocol/go-sdk` puts documentation contributions under CC-BY-4.0. Documentation is not compiled into `terragrunt`, so neither term is in the label.

The default image adds OpenTofu (see [randomcontainers/opentofu](https://github.com/randomcontainers/opentofu#licenses) for its license, its source and its license files), Git from Ubuntu or Alpine (GPL-2.0-only) and the OpenSSH client (`SSH-OpenSSH AND MIT`); see [randomcontainers/openssh-client](https://github.com/randomcontainers/openssh-client#licenses) for the OpenSSH sources and license files. Git and the other packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

MPL-2.0, and the GPL for Git in the default image, require whoever distributes a binary to make its source available to its recipients. The source code for each image:

- Terragrunt: every version has a GitHub release in this repository, named `v<version>`, with `terragrunt-<version>.tar.gz`, an archive of the commit its tag points to. The build applies no patches.
- Go modules under MPL-2.0: the module zips in `/usr/local/share/randomcontainers/terragrunt/mpl-sources/`, downloaded from the Go module proxy and checked against `go.sum`. `/usr/local/share/randomcontainers/terragrunt/source` lists the Terragrunt commit, these zips and the release.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every build flag.
- Ubuntu packages, including Git in the default image: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages, including Git in the default image: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
