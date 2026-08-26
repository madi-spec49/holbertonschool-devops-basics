# Select a Compatible Minimal Base Image — Decision

## Candidate sizes and base references

| Candidate | Base image | Size (bytes) |
|---|---|---|
| base-lab:ubuntu | ubuntu:24.04 | 28,890,846 |
| base-lab:debian-slim | debian:12-slim | 28,120,333 |
| base-lab:alpine | alpine:3.22 | 4,123,674 |

All three candidates printed the required output: {"runtime":"posix-shell","status":"ok"}

## Selected base

alpine:3.22 — roughly 7x smaller than either Debian-slim or Ubuntu.

runtime-requirements.md requires: a Linux image, a POSIX-compatible /bin/sh, no
glibc-specific native extension, no package manager operation after the image is built, no
interactive debugging shell requirement in production, and a non-root configured runtime
user. Alpine satisfies every one of these:

- It is Linux-based and ships a POSIX-compliant /bin/sh via BusyBox.
- The application is a plain shell script with no compiled native extensions, so Alpine's use
  of musl instead of glibc is not a problem.
- The Dockerfile never runs apk, apt, or any package manager after the base image is
  pulled, so Alpine's minimal package set is not a limitation here.
- Nothing in the application needs an interactive debugging shell in production.
- A non-root user (65532:65532) was added explicitly in Dockerfile.selected.

Since Alpine meets every stated requirement and is the smallest of the three candidates, it
is the correct choice under the instructions.

## base-lab:selected verification

docker build -f Dockerfile.selected -t base-lab:selected .
docker run --rm base-lab:selected
docker image inspect base-lab:selected --format '{{.Config.User}}'

Output: {"runtime":"posix-shell","status":"ok"}
65532:65532

## When Debian slim would be safer than Alpine despite being larger

If the application instead depended on a compiled binary or native extension linked against
glibc, Alpine's musl libc could cause the binary to fail to run, behave subtly differently,
or require rebuilding against musl. Debian slim ships glibc and a fuller set of standard
system libraries, so it would be the safer choice for any workload with real, non-trivial
native/OS dependencies — even though it is roughly 7x larger than Alpine here.
runtime-requirements.md explicitly warns about exactly this case: applications that require
glibc or a vendor-supported Debian package should not assume Alpine is correct.

## Versioned tag vs. digest for the base image reference

A versioned tag (e.g. alpine:3.22) is useful for control: it is human-readable, makes the
intended major/minor version obvious in the Dockerfile, and can be intentionally bumped as
part of a deliberate, reviewable change.

However, a tag is not immutable — the same tag name can be re-pushed by the image publisher
to point at a different underlying image, so alpine:3.22 today is not guaranteed to be
byte-for-byte the same image as alpine:3.22 next month. A digest
(alpine:3.22@sha256:...) pins the build to one exact, content-addressed image and cannot be
silently changed. For a truly immutable, reproducible base image reference, the reference
must include the digest; the tag alone only gives an approximate, human-friendly version,
not a guarantee.
