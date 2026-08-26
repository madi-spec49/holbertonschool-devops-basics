# Preserve the Dependency Cache — Observations

## Unoptimized Dockerfile: source-only change invalidates the dependency layer

Dockerfile.unoptimized copies everything with a single COPY . . before running
npm ci --omit=dev. After adding a comment to src/server.js and rebuilding with the same
command (no --no-cache), the build log showed:

#9 [3/4] COPY . .
#9 DONE 0.0s
#10 [4/4] RUN npm ci --omit=dev && node -e "setTimeout(() => {}, 3000)"
#10 DONE 3.7s

Neither COPY . . nor RUN npm ci reported CACHED — both re-ran in full, even though
nothing about the dependencies changed.

Why npm ci re-runs: Docker's layer cache is invalidated instruction-by-instruction based on
the content each instruction consumes. COPY . . copies the entire build context into a
single layer; changing src/server.js changes that layer's hash, invalidating it. Every
instruction after an invalidated layer is invalidated too, regardless of whether it actually
depends on the changed file — so npm ci, which only truly depends on package.json,
package-lock.json, and packages/message-format/, is forced to re-run anyway.

## Cached Dockerfile: source-only change preserves the dependency layer

Dockerfile.cached separates dependency manifests and the local message-format package from
the rest of the source, and installs dependencies before copying src/ and test/:

COPY package.json package-lock.json ./
COPY packages/message-format/ ./packages/message-format/
RUN npm ci --omit=dev && node -e "setTimeout(() => {}, 3000)"
COPY src/ ./src/
COPY test/ ./test/

After building cache-lab:cached once, a comment-only change was made to src/server.js and
the same build command was run again (no --no-cache):

#13 [5/7] RUN npm ci --omit=dev && node -e "setTimeout(() => {}, 3000)"
#13 CACHED

The dependency-install layer reported CACHED on the second build. Only the layers copying
src/ and test/ were re-run — the expensive npm ci --omit=dev step, including its simulated
3-second delay, was skipped entirely.

## Runtime verification

docker network create cache-lab-net
docker run --rm -d --name cache-lab --network cache-lab-net cache-lab:cached
docker run --rm --network cache-lab-net busybox:1.37 sh -c '...'

Response: {"service":"cache-lab","status":"ok"}

The container was removed and the network torn down afterward.

## Summary

| Dockerfile | Change made | npm ci --omit=dev result |
|---|---|---|
| Dockerfile.unoptimized | comment in src/server.js | re-ran (3.7s, not cached) |
| Dockerfile.cached | comment in src/server.js | CACHED |

Ordering COPY instructions so that only the files an instruction actually depends on can
invalidate it — manifests and the local dependency package before npm ci, application source
and tests after — lets a source-only change invalidate just the source-copy and later
layers, while the dependency-install layer is correctly reused.
