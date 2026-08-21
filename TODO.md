# TODO — App byte movement

Status: implemented; automated and fresh-source desktop byte-flow QA complete,
including native open and save picker dialogs.
Owner: Codex.
Citations re-verified against the working tree on 2026-08-20.

Implementation and focused verification completed on 2026-08-20. The primary Dev
profile was rebuilt and restarted from the final bundle, and a fresh QA App tab
successfully exercised full and ranged local reads, URL revocation, generated and
stored-file uploads, an atomic download, transfer progress, a native open picker,
and a native save picker. The open picker granted and streamed a 74-byte Markdown
fixture; the save picker granted a writable handle for a new filename in `/tmp`
without creating the destination eagerly.

## Problem

An App cannot move bulk bytes. Picking a 400 MB video and playing it is not
expressible today; neither is uploading a picked file, nor reading a file the
App itself wrote to its own storage.

The cause is that Apps have two disjoint byte namespaces with no edge between
them:

- **Handle space** — user-picked handles from `files.pick`. Has parse verbs
  (`readText`, `readBinary`), no transport verbs.
- **Storage space** — the App×Space durable root. Has transport verbs
  (`fetchToFile`, `uploadFromFile`), no read verb at all.

Six edges are missing: handle→network, network→handle, storage→renderer,
storage→handle, handle→storage, handle→composer.

The obvious fix — drop the bespoke vocabulary and let Apps use standard web
APIs — was investigated and **does not work**. See Findings.

## Findings (verified)

Measured on Electron 40.10.6 / Chromium 144.0.7559.60 with a harness that
mirrored the real topology: privileged `penkra-app://` scheme, `protocol.handle`
on a partition session, cross-origin child iframe carrying the exact `sandbox`
attribute from `apps/web/src/components/chat/AppDockPane.tsx:185-190`, and the
real CSP from `apps/desktop/src/appPackageProtocol.ts:12-24`.

**F1 — The File System Access API is unreachable from an App.**

```
SecurityError: Failed to execute 'showOpenFilePicker' on 'Window':
Cross origin sub frames aren't allowed to show a file picker.
```

Not a policy gate — Chromium hard-checks frame ancestry, and there is no
permissions-policy token to delegate. `penkra-app://` is `standard: true,
secure: true` (`desktopProtocolSchemes.ts:43-48`) so the secure-context
requirement is satisfied; the scheme is not the blocker, the frame is. Apps are
only ever iframes: `grep -n "WebContentsView|BrowserView" apps/desktop/src/main.ts`
returns nothing.

Consequence: `files.pick` is not a redundant reinvention of
`showOpenFilePicker`. It is the only way an App can obtain a user-chosen file.
It stays.

**F2 — Streaming request bodies reach the main process intact.**

```
handler A: method=POST bodyStream=true bytesRead=3145728   fetch(Blob)
handler B: method=POST bodyStream=true bytesRead=2097152   fetch(ReadableStream, duplex:"half")
handler C: method=POST bodyStream=true bytesRead=6291456   XMLHttpRequest
```

`protocol.handle` receives a standard `Request` whose `body` is a readable
stream, byte-exact, with no size cap. Both historical blockers are closed
(electron#39658, electron#41872).

**F3 — Native upload progress does not fire on the custom scheme.**

`upload.onprogress` produced zero ticks. A control XHR to a loopback HTTP server
from the same iframe did fire, so this is the scheme, not the harness.

This is fine, and arguably correct. A local progress event would measure handoff
to the local protocol handler, not transfer to the remote — it would report
"done" while the real upload had not started. Progress must be host-emitted.

**F4 — The local backend is unauthenticated.**

`apps/server/src/config.ts:43` (`remoteAccessPolicyError`), lines 53-54:

```ts
const isPubliclyExposed = isRemoteBind || Boolean(config.publicUrl);
if (!isPubliclyExposed) return null; // no auth token required
```

With `effectServer.ts:173` defaulting to `host: "127.0.0.1"`, any SSRF reachable
from an App renderer is a direct, credential-free attack on the whole Penkra
API. **The host must name and validate every network destination. This does not
relax.**

**F5 — No guard needs to change.**

Because the transport rides the App's _own origin_:

- `connect-src 'self'` (`appPackageProtocol.ts:19`) already permits it.
- The `onBeforeRequest` cancel filter (`appSessionManager.ts:218-220`) routes
  through `belongsToAssignedOrigin`; same-origin already passes.
- `protocol.handle(PENKRA_APP_SCHEME, ...)` is already registered per partition
  through a mutable target (`appSessionManager.ts:113`).
- A reserved-path convention already exists —
  `APP_FRAME_RUNTIME_PATH = "/.penkra/runtime.js"`, intercepted before package
  resolution (`appPackageProtocol.ts:51-56`).

This is a new branch in an existing handler. No new scheme, no new permission,
no CSP change. Constraint to respect: `form-action 'none'` means no `<form>`
POST — `fetch`/XHR only.

Also checked and **not** a gap: `storage.fetchToFile` and `uploadFromFile` are
gated on `network-fetch` at `main.ts:502-514`, same as `network.fetch`.

## Design

**Give bytes a same-origin URL.** Once a picked file or a stored file has a URL,
the App hands it to `fetch()` or `<video src>` and writes ordinary web code. Most
of the missing edges close without new byte vocabulary.

Two concepts total:

1. `files.open` / `storage.open` → a same-origin URL backed by a ranged GET.
2. `transfer.*` → host-named destinations, because of F4.

### 1. `files.open` / `storage.open`

```ts
files.open(handleId: string, relativePath?: string): Promise<string>;
storage.open(path: string): Promise<string>;
```

Returns `penkra-app://<origin>/.penkra/blob/<token>`. The handler serves it with
`Accept-Ranges: bytes` and honours `Range`, so the browser streams and seeks
natively. Zero bytes cross the bridge.

Lifetime follows `URL.createObjectURL` (agreed: "whatever's standard"): the token
lives until explicitly revoked or the owning tab unloads.

```ts
files.closeUrl(url: string): Promise<void>;
storage.closeUrl(url: string): Promise<void>;
```

Named `closeUrl` rather than `revoke` because `files.revoke(handleId)` already
exists (`packages/sdk/src/runtime.ts:193`) and means something else — revoking
the _handle grant_. Reusing the word would be a trap.

Tokens are unguessable, bound to the issuing App×Space partition, and rejected if
presented from another origin.

### 2. `transfer.*`

```ts
transfer.begin(input: {
  url: string;
  method?: "POST" | "PUT" | "PATCH";
  headers?: Record<string, string>;
}): Promise<{ id: string; endpoint: string }>;

transfer.send(input: {
  url: string;
  method?: "POST" | "PUT" | "PATCH";
  headers?: Record<string, string>;
  from: { handleId: string; relativePath?: string } | { storage: string };
  field?: string;
}): Promise<{ id: string; status: number; headers: Record<string, string>; body: string }>;

transfer.receive(input: {
  url: string;
  method?: "GET" | "POST";
  headers?: Record<string, string>;
  body?: string;
  to: { storage: string } | { handleId: string; relativePath?: string };
}): Promise<{ id: string; bytes: number; sha256: string }>;

transfer.onProgress(listener: (event: AppTransferProgressEvent) => void): () => void;
```

```ts
interface AppTransferProgressEvent {
  id: string;
  phase: "uploading" | "downloading";
  movedBytes: number;
  totalBytes: number | null; // null when the remote sends no length
}
```

`begin` validates and pins the destination through the existing
`appNetworkFetch.ts` path, then returns a single-use same-origin endpoint. The
handler pipes `request.body` straight to the pinned socket (F2). All three verbs
are gated on `network-fetch`.

Progress is emitted host-side (F3), following the `browser.download` precedent —
`main.ts:1018` emits, `appFrameRuntime.ts:148-155` subscribes.

### 3. Save picker

`files.pick` gains a save kind, since a save picker hits the same cross-origin
block as the open picker (F1) and must be host-side:

```ts
files.pick(kind: "file" | "directory" | "save", options?: {
  suggestedName?: string;
}): Promise<AppScopedFileHandle | null>;
```

The returned handle is writable and feeds `transfer.receive({ to: { handleId } })`
or the existing chunked write path.

## Worked examples

Scrub a picked video — the motivating case, impossible today:

```ts
import { files } from "@penkra/sdk";

const handle = await files.pick("file");
if (handle) video.src = await files.open(handle.id);
```

Get real bytes with no 1 MiB cap and no chunk loop:

```ts
const blob = await (await fetch(await files.open(handle.id))).blob();
```

Show something the App wrote to its own storage — the storage→renderer edge,
which has no verb at all today:

```ts
import { storage } from "@penkra/sdk";
img.src = await storage.open("thumbs/cover.png");
```

Upload bytes the App generated:

```ts
import { transfer } from "@penkra/sdk";

const { endpoint } = await transfer.begin({
  url: "https://api.example.com/v1/documents",
  method: "POST",
  headers: { "content-type": "application/json" },
});
const res = await fetch(endpoint, { method: "POST", body: docBlob });
```

Upload a picked file the App must never hold:

```ts
const res = await transfer.send({
  url: "https://api.example.com/v1/uploads",
  method: "POST",
  from: { handleId: handle.id },
  field: "file",
});
```

Download to a location the user chooses:

```ts
const target = await files.pick("save", { suggestedName: "export.pen" });
if (target) await transfer.receive({ url: assetUrl, to: { handleId: target.id } });
```

Report honest progress:

```ts
const stop = transfer.onProgress((e) => {
  bar.value = e.totalBytes ? e.movedBytes / e.totalBytes : null;
});
```

## Deletions

Clean cuts, no deprecation window — the repo is early WIP. A repo-wide grep for
`uploadFromFile|fetchToFile` finds **no App consumer at all** — `penkra-apps/`
has zero hits. Every hit is host-side plumbing, SDK surface, a test mock, or
docs. The cut is cheap.

- **Delete** `storage.uploadFromFile` → `transfer.send({ from: { storage } })`.
- **Delete** `storage.fetchToFile` → `transfer.receive({ to: { storage } })`.
- **Keep** `files.readText` / `readBinary` / `writeText`, demoted to what they
  should always have been: small config-ish reads. They stop being the
  load-bearing path for bulk bytes.
- **Keep** the chunked write path (`appScopedFileWriteStore.ts`) — it is the only
  atomic writer and `transfer.receive` should reuse its commit semantics.

PR #2's 32 MB bump stays correct and stops being the thing standing between a
user and a video.

## Work items

1. [x] **Blob URL registry** — new `appBlobUrlRegistry.ts`. Token mint/lookup/revoke,
       partition-bound, tab-scoped lifetime. Unguessable tokens.
2. [x] **Protocol handler branch** — extend `appPackageProtocol.ts` to intercept
       `/.penkra/blob/<token>` before package resolution, alongside the existing
       `/.penkra/runtime.js` branch. Ranged GET, `Accept-Ranges: bytes`, correct
       206/416 handling.
3. [x] **Transfer endpoint branch** — intercept `/.penkra/transfer/<ticket>`. Pipe
       `request.body` to the pinned socket. Single-use tickets; reject replay.
4. [x] **Transfer service** — new `appTransfer.ts`. Owns ticket lifecycle, reuses
       `appNetworkFetch.ts` destination pinning and the `appStorage.ts` backpressure
       loop. Emits progress frame events.
5. [x] **IPC + runtime plumbing** — `main.ts` method table, `appFrameRuntime.ts`
       `onTransferProgress`, `packages/sdk/src/runtime.ts` surface.
6. [x] **Save picker** — extend `files.pick` with the `"save"` kind and a writable
       handle grant.
7. [x] **Deletions** — remove `storage.uploadFromFile` / `fetchToFile`. Verified
       surface is host-side only, no App migration needed. Six files:
       `packages/sdk/src/runtime.ts` (227, 239, 412, 414),
       `packages/sdk/src/runtime.test.ts` (62, 64),
       `apps/desktop/src/main.ts` (502, 519-526, 5887),
       `apps/desktop/src/appPreloadRuntime.ts` (134-144),
       `apps/desktop/src/appStorage.ts`,
       `docs/app-development.md` (300, 308).
8. [x] **Docs** — see below.

## Verification

- Desktop package: full suite passed (130 files passed, 1 skipped; 821 tests
  passed, 5 skipped).
- SDK package: full suite passed (4 files, 19 tests).
- Final focused rerun passed: 7 desktop files / 34 tests and 2 SDK files / 5
  tests.
- Desktop bundle build completed successfully from the current source.
- Fresh Penkra Dev startup/render passed. In a newly opened App tab from the final
  bundle, manual QA passed for a full local read, ranged local read, URL close,
  512 KiB generated-body upload, 10-byte storage upload, 1 KiB atomic download,
  upload/download progress, a native open picker that read the selected 74-byte
  fixture, and a native save picker that returned a writable handle for the exact
  requested filename.
- The final repository-wide release gate includes formatting, lint, typecheck,
  workspace and browser tests, the desktop pipeline, and release smoke checks.

## Docs

`docs/app-development.md` now documents the URL-based byte model, including the
1 MiB RPC boundary, worked byte-movement examples, why Apps must use host-side
pickers, host-named transfer destinations, and `transfer.onProgress`. Its
`storage.*` reference was updated for the clean-cut deletions.

`docs/app-runtime-security.md` now records the destination-validation rationale
and blob-token threat model.

`docs/app-runtime-security.md` should record the F4 rationale and the
blob-token threat model.

## Resolved questions and follow-ups

- `transfer.receive({ to: { handleId } })` uses the same atomic guarantees as
  the scoped writer (temporary file + fsync + rename + SHA-256), implemented in
  the transfer service so the response can stream directly to disk.
- No separate method-level version gate was introduced. App compatibility
  remains declared through the existing manifest `compatibility.penkra` range.
- `composer.stage` still reads attachments via `FSP.readFile` at a 256 MiB cap
  (`appStorage.ts`, `readComposerAttachment`). Should it move to the blob-URL
  path? Out of scope here; flagged.
- `files.writeText` is **not** atomic (bare `writeFile`, no fsync, no
  temp+rename) while the chunked path is. Inconsistent; worth fixing or
  documenting independently of this work.

## Decided

- Blob URL lifetime: standard `createObjectURL` semantics — explicit revoke,
  auto-release on tab unload.
- Save picker: in scope, not deferred.
- Deletions: clean cuts, no deprecation window.
- Storage quotas: not adding any. Only the existing 512 MiB free-disk floor
  (`DEFAULT_MIN_FREE_BYTES`) applies.
