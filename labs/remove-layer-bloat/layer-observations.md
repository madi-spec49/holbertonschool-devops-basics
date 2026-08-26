# Remove Layer Bloat at Its Source — Observations

## Checksum verification

Both images print the identical checksum:
a0e8bdd8a312de8e45d2cea454dee228ede781730bfc321d96a6fced1b634090

docker run --rm layer-lab:unoptimized -> a0e8bdd8a312de8e45d2cea454dee228ede781730bfc321d96a6fced1b634090
docker run --rm layer-lab:optimized   -> a0e8bdd8a312de8e45d2cea454dee228ede781730bfc321d96a6fced1b634090

## Image sizes

docker image inspect layer-lab:unoptimized --format '{{.Size}}'  -> 10417781
docker image inspect layer-lab:optimized   --format '{{.Size}}'  -> 4123338

Byte difference: 10,417,781 - 4,123,338 = 6,294,443 bytes (~6.0 MiB) - well above the
required 5 MiB (5,242,880 bytes) minimum.

## docker image history comparison

Unoptimized (3 separate RUN instructions):
RUN rm -f /tmp/build-payload.bin                              8.19kB
RUN sha256sum /tmp/build-payload.bin | cut -d ' ' -f 1 ...     8.19kB
RUN cp /mnt/build-payload.bin /tmp/build-payload.bin           6.3MB   <-- payload retained here

Optimized (single merged RUN instruction):
RUN cp /mnt/build-payload.bin /tmp/... && sha256sum ... && rm ...    8.19kB

The layer that retains /tmp/build-payload.bin in the unoptimized image is the
RUN cp /mnt/build-payload.bin /tmp/build-payload.bin layer (6.3MB). Even though a later
layer runs rm -f /tmp/build-payload.bin, that later layer is only 8.19kB - it does not
shrink the earlier 6.3MB layer at all.

## Why the later rm hides the file but cannot remove its bytes

A Docker image is a stack of read-only, immutable layers. Each RUN instruction produces its
own layer as a diff against the layer below it, and once written, a layer's diff is never
modified or rewritten by any later instruction - it is content-addressed and frozen.

When RUN cp /mnt/build-payload.bin /tmp/build-payload.bin runs, it writes the file's bytes
into that layer's diff. That diff is finalized and stored as soon as the RUN step completes;
nothing that happens afterward can reach back and edit it.

When the later RUN rm -f /tmp/build-payload.bin runs, it does not (and cannot) reach into
the earlier layer and delete those bytes. Instead, it records a "whiteout" marker in its own
layer's diff - a small file-system entry that tells the container runtime "treat this path as
absent" when the layers are stacked and merged into the container's final view of the
filesystem (the "merged filesystem," e.g. via OverlayFS). That whiteout is tiny, which is why
the rm layer only adds 8.19kB.

The result is a filesystem-view illusion: docker run and anything running inside the
container sees a merged filesystem where /tmp/build-payload.bin does not exist, because the
whiteout hides it. But docker image inspect/.Size and docker save still account for every
layer's stored bytes, including the 6.3MB cp layer - because that layer is still physically
part of the image, unpacked and stored on disk, and still gets pulled/pushed with the image.
Deleting a file in a later layer changes what the final filesystem looks like; it does not
change what earlier, already-committed layers contain.

The optimized Dockerfile avoids this entirely by never letting the payload's bytes become
part of a layer that survives: cp, sha256sum, and rm all happen inside the same RUN
instruction, so only the single resulting diff - after the file has already been copied,
hashed, and deleted - is what gets committed as that layer. The payload's bytes never
outlive the instruction that created them, so no earlier layer is left holding data that a
later layer then has to hide.
