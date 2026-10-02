# imagemagick

Container images with [ImageMagick](https://imagemagick.org/) 7, compiled from the upstream release against the image libraries of Ubuntu or Alpine. libheif, which ImageMagick uses for HEIC and AVIF, and the HEVC decoder libde265 are compiled from their upstream releases too. The default image also contains Ghostscript, which ImageMagick needs to read PDF, PostScript and EPS files. The images are rebuilt when ImageMagick publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the ImageMagick project. Report problems with the image in this repository and problems with ImageMagick itself [upstream](https://github.com/ImageMagick/ImageMagick/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/imagemagick input.png -resize 50% output.webp
```

Render the first page of a PDF at 150 dpi. Reading a PDF needs Ghostscript, which is in the default image but not in `slim`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/imagemagick -density 150 "input.pdf[0]" page-1.png
```

The entrypoint runs `magick` under `tini`, so the arguments go to `magick`. The older command names work as the first argument:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/imagemagick identify photo.heic

docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/imagemagick montage -geometry 240x240+4+4 *.jpg contact-sheet.jpg
```

`mogrify` changes files in place. Give it `-path` to write the results to another directory.

## What is in the image

| Area | Details |
|---|---|
| Build | ImageMagick 7, Q16 with HDRI, OpenMP, coders as loadable modules |
| Raster formats | JPEG (libjpeg-turbo), PNG, GIF, TIFF, WebP, JPEG XL, JPEG 2000 (OpenJPEG), AVIF (libheif with aom and dav1d), HEIC (read only, libheif with libde265) |
| Camera raw | DNG, CR2, CR3, NEF, ARW and the other formats LibRaw reads |
| Vector | SVG through ImageMagick's built-in renderer and libxml2 |
| Documents | PDF, PostScript and EPS. Both images write them; reading them needs Ghostscript, which only the default image has |
| Color and compression | Little CMS 2, zlib, zstd, xz, bzip2 |
| Text | FreeType, fontconfig and the DejaVu fonts |

The Ubuntu and Alpine libde265 packages lack upstream security fixes, so libde265 and libheif are compiled here instead. The codecs are built into libheif, which loads no plugins; aom and dav1d come from the distro.

Not included: X11 (`display`, `animate` and `import` exit with an error), librsvg, Pango, OpenEXR, FFTW, HEIC encoding (libheif is built without an HEVC encoder), video formats that ImageMagick hands to `ffmpeg`, the Perl and C++ bindings, and the C headers. The configure flags, the libheif and libde265 versions and the full `magick -version` output are in `/usr/local/share/randomcontainers/imagemagick/buildinfo`.

### Fonts

Text drawn with `label:`, `caption:`, `-annotate` or SVG uses DejaVu Sans unless you pick a font. Helvetica and the SVG family `sans-serif` map to DejaVu Sans, Times-Roman and `serif` to DejaVu Serif, and Courier and `monospace` to DejaVu Sans Mono, each with a bold form. `magick -list font` shows every font by name. To use another font, pass its file, for example `-font /work/fonts/Inter.ttf`, or install more font packages in an image built on `slim`.

## Default or slim

Use the default image (`latest`) for general work: it also reads PDF, PostScript and EPS files, with no extra setup. `slim` has ImageMagick and the libraries it needs, without Ghostscript; use it to build your own image, or when you only work with raster formats and SVG. The default images of [ExifTool](https://github.com/randomcontainers/exiftool) and [jpegoptim](https://github.com/randomcontainers/jpegoptim) include the `slim` build.

The default image includes Ghostscript, which is licensed under the GNU Affero General Public License (AGPL-3.0-or-later). Use `slim` if your policy excludes AGPL. The default image is also published as `ghcr.io/randomcontainers/imagemagick-ghostscript`, built in the [imagemagick-ghostscript](https://github.com/randomcontainers/imagemagick-ghostscript) repository with the same contents and a different digest.

## Tags

`<version>` is an ImageMagick release such as `7.1.2-31`. `<x.y.z>`, `<x.y>` and `<x>` are its shorter forms, `7.1.2`, `7.1` and `7`, and follow the newest release in that series.

| Default (with Ghostscript) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<x.y.z>`, `<x.y>`, `<x>` | `slim`, `<version>-slim`, `<x.y.z>-slim`, `<x.y>-slim`, `<x>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<x.y.z>-ubuntu`, `<x.y>-ubuntu`, `<x>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<x.y.z>-slim-ubuntu`, `<x.y>-slim-ubuntu`, `<x>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<x.y.z>-alpine`, `<x.y>-alpine`, `<x>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<x.y.z>-slim-alpine`, `<x.y>-slim-alpine`, `<x>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current ImageMagick version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Security policy

ImageMagick reads its security policy from `/usr/local/etc/ImageMagick-7/policy.xml`. The images use [`policy.xml`](policy.xml) from this repository, which is ImageMagick's `open` policy with two rules added:

- Indirect reads are blocked. `@file` arguments such as `label:@notes.txt`, or `@files.txt` to read a list of input names, fail with `not authorized`. Put the text or the file names on the command line instead.
- The MSL coder (ImageMagick Scripting Language) is disabled.

PDF, PostScript, EPS and SVG stay enabled, and ImageMagick starts Ghostscript with `-dSAFER`. Print the active policy with `magick -list policy`.

If you convert files from untrusted sources, for example uploads in a web service, mount a stricter policy over the default. ImageMagick publishes a [`websafe` policy](https://github.com/ImageMagick/ImageMagick/blob/main/config/policy-websafe.xml) that allows only common web formats and sets resource limits; [the security policy guide](https://imagemagick.org/script/security-policy.php) explains the options.

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -v "$PWD/policy-websafe.xml:/usr/local/etc/ImageMagick-7/policy.xml:ro" \
  ghcr.io/randomcontainers/imagemagick upload.jpg -resize 800x800 thumbnail.webp
```

## Memory, disk and threads

The policy sets no resource limits, because ImageMagick treats a limit in the policy as a ceiling that `-limit` and the `MAGICK_*_LIMIT` variables cannot raise. Set limits when you run the container instead:

- ImageMagick sizes its pixel cache from the host's memory and does not see a `docker run --memory` limit. Pair `--memory` with lower ImageMagick limits so that large images move to a disk cache before the container is killed, for example `--memory 2g` with `-limit memory 1GiB -limit map 1GiB`, or the variables `MAGICK_MEMORY_LIMIT` and `MAGICK_MAP_LIMIT`.
- The disk cache is written to `/tmp` inside the container. Mount a volume and set `MAGICK_TEMPORARY_PATH` to it for large jobs, and cap it with `-limit disk 20GiB`.
- OpenMP starts one thread per host CPU, including under `--cpus`. Set `MAGICK_THREAD_LIMIT` (or `-limit thread`) to the number of CPUs you give the container.

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --memory 2g --cpus 2 -e MAGICK_THREAD_LIMIT=2 \
  ghcr.io/randomcontainers/imagemagick -limit memory 1GiB -limit map 1GiB \
  scan.tif -resize 50% scan-small.tif
```

`magick -list resource` prints the limits in effect.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no Ghostscript, so Ghostscript updates do not rebuild it. The packages ImageMagick needs are listed in `/usr/local/share/randomcontainers/imagemagick/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/imagemagick:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends fonts-noto-core \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache font-noto`. The entrypoint is `["tini", "--", "magick"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new ImageMagick releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/imagemagick:latest \
  --repo randomcontainers/imagemagick --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/imagemagick-ghostscript` are built in that repository, so verify them with `--repo randomcontainers/imagemagick-ghostscript` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/imagemagick:latest --format '{{ json .SBOM }}'
```

libheif and libde265 are compiled here, so they are not in the SBOM and image scanners do not check them. Their versions are in `/usr/local/share/randomcontainers/imagemagick/buildinfo`.

The build downloads the source archives that ImageMagick, libheif and libde265 attach to their GitHub releases (for ImageMagick `ImageMagick-<version>.tar.xz`, not the automatic tag archive) and checks each one against the SHA-256 recorded in `package.yml` before compiling.

## Updates

The project checks the [ImageMagick/ImageMagick](https://github.com/ImageMagick/ImageMagick) releases every 15 minutes and follows the 7.x series. A release is picked up once it is 24 hours old. The new version and the SHA-256 of its source archive are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built. ImageMagick 6 is not built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Ghostscript image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

libheif and libde265 are pinned to a version in `package.yml` and do not follow their upstream releases automatically. Updating one is a commit to `package.yml`, which rebuilds the images of the current ImageMagick version.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  --build-arg LIBDE265_VERSION=<version> \
  --build-arg LIBDE265_SHA256=<sha256> \
  --build-arg LIBHEIF_VERSION=<version> \
  --build-arg LIBHEIF_SHA256=<sha256> \
  -t imagemagick:local .
```

The `LIBDE265_*` and `LIBHEIF_*` values are the `version` and `sha256` of the `extra-artifacts` entries in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

ImageMagick is distributed under the [ImageMagick License](https://imagemagick.org/license/) (SPDX `ImageMagick`). Its license and notice files are in `/usr/local/share/randomcontainers/imagemagick/licenses/`.

libheif and libde265 are licensed under the GNU Lesser General Public License, version 3 or later (LGPL-3.0-or-later), so the image's license label is `ImageMagick AND LGPL-3.0-or-later`. Their license files are `COPYING.libheif` and `COPYING.libde265` in the same directory. The libraries and fonts from Ubuntu or Alpine keep their own licenses, for example LGPL for LibRaw, BSD-2-Clause for aom and dav1d, and the DejaVu font license. The SBOM lists them.

The corresponding source for each image:

- ImageMagick, libheif and libde265: every ImageMagick version has a GitHub release in this repository, named `v<version>`, with the exact `ImageMagick-<version>.tar.xz` and the libheif and libde265 archives that were compiled. The build applies no patches. It only replaces ImageMagick's font map, `config/type-dejavu.xml.in`, with [the copy in this repository](type-dejavu.xml.in). `/usr/local/share/randomcontainers/imagemagick/source` lists that release and the download URLs of the three archives. When libheif or libde265 is updated, the release also keeps the archive of the earlier version.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure and CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The default image adds Ghostscript, licensed under AGPL-3.0-or-later; see [randomcontainers/ghostscript](https://github.com/randomcontainers/ghostscript) for its sources and license files.

Some formats, such as HEIC, may be covered by patents in some countries. Check what applies where you use them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
