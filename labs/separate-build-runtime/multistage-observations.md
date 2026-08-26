# Separate Build and Runtime with Multiple Stages — Observations

## Single-stage baseline

docker build -f Dockerfile.single-stage -t multistage-lab:single .
docker run --rm multistage-lab:single
docker image inspect multistage-lab:single --format '{{.Size}}'

Output: {"service":"greeter","status":"ok"}

Size: 76,211,788 bytes (~76.2 MB) - includes the full golang:1.25-alpine toolchain (Go
compiler, standard library, source tree) baked into the final image, even though none of it
is needed to run the compiled binary.

## Multi-stage build

Dockerfile.multistage uses a named build stage on golang:1.25-alpine that downloads
dependencies (go mod download) before the source is copied, runs go test ./..., and builds a
statically linked binary with CGO_ENABLED=0. The final stage is FROM scratch and copies in
only the compiled binary via COPY --from=build, with USER 65532:65532 and an exec-form
ENTRYPOINT.

docker build -f Dockerfile.multistage -t multistage-lab:optimized .
docker run --rm multistage-lab:optimized

Output: {"service":"greeter","status":"ok"} - identical to the single-stage image.

## Size comparison

docker image inspect multistage-lab:optimized --format '{{.Size}}'  -> 1,271,837 bytes

| Image | Size |
|---|---|
| multistage-lab:single | 76,211,788 bytes |
| multistage-lab:optimized | 1,271,837 bytes |

Difference: 74,939,951 bytes (~71.5 MB) smaller - a ~98% size reduction. The final image
contains only the compiled Go binary: no Go compiler, no standard library source, no /src
tree, and no shell.

## Configured user

docker image inspect multistage-lab:optimized --format '{{.Config.User}}'  -> 65532:65532

The runtime stage runs as a non-root, numeric user/group.

## Shell override attempt

docker run --rm --entrypoint /bin/sh multistage-lab:optimized

Result: exec: "/bin/sh": stat /bin/sh: no such file or directory

## Why the failed shell command is expected

The final stage is built FROM scratch, which starts with an empty filesystem - no shell, no
package manager, no C library, no /bin directory at all. Only the single statically linked
binary (/usr/local/bin/greeter, built with CGO_ENABLED=0 so it has no dynamic library
dependencies) was copied in. /bin/sh was never part of any layer in this image, so
attempting to exec it fails at the OS level before the container process can even start -
there is nothing on disk at that path to run.

This is the intended security and size benefit of separating build and runtime with
multi-stage builds: an attacker or operator who compromises or inspects a running container
cannot drop into an interactive shell, install tools, or explore the filesystem, because
none of that tooling exists in the image.

## Why this does not replace functional testing of the application binary

The failed shell command only proves that no shell is present - it says nothing about
whether the application binary itself works correctly. Confidence that the program behaves
correctly comes from two other things that already happened earlier in the pipeline:

1. go test ./... ran inside the build stage against real source, verifying message.JSON()
   returns the exact expected string; the build would have failed if any test failed.
2. docker run --rm multistage-lab:optimized (using the image's normal ENTRYPOINT, not an
   overridden one) was executed directly and produced the correct
   {"service":"greeter","status":"ok"} output.

The absence of a shell is a property of the image's contents, verified by trying to invoke a
binary that isn't there. Functional correctness of the application is a property of the
program's behavior, verified by unit tests and by actually running the program's real
entrypoint and checking its output. One check does not substitute for the other: an image
could have no shell and still ship a broken binary, or could ship a shell and still run a
correct binary. Both checks are necessary and neither implies the other.
