# TODO — Agent-facing writing and command surface

Status: planned. No code changed yet.
Owner: unassigned.
Evidence gathered against the working tree on 2026-08-22.
Supersedes the completed App byte-movement record, deleted 2026-08-22 on the
owner's instruction.

---

## Problem

Penkra tells agents what it is, what is installed, and how to act through a body
of prose that nobody designed as a whole. It accreted: a bullet added to fix one
failure, a provider-specific line added while debugging that provider, a
prohibition added after an agent did something surprising. The result works often
enough to hide that it is not a system.

Three failures follow from that, and all three are reproducible.

**Agents skip discovery and act on prior belief.** The injected policy says to run
`penkra --help` "when the relevant hierarchy is unknown." That conditional keys on
the agent's own confidence, so it fails exactly when the agent is confidently
wrong. A session tasked with "create a new Canvas design" spent its first several
turns inside an unrelated MCP server while a `canvas` App with fourteen document
operations sat one command away, enabled, in the same Space.

**The prose contradicts the implementation.** The policy instructs agents to run
`penkra apps list` to discover Apps. `penkra --help` already returns the entire
App catalog with every operation. Two commands are presented as sequential steps
when one is strictly contained in the other.

**Load-bearing nouns are never defined.** `docs/app-development.md` uses "Space"
more than thirty times and never says what one is. Line 204 corrects a
misunderstanding of the term without ever introducing it. The same holds for
Thread, Project, operation, controller, installation, and tab. Every paragraph
about isolation, permission, and storage is built on words the reader must guess.

Underneath all three sits a shape problem. Apps have a good shape: a document
(`INSTRUCTIONS.md`), a command surface, and nested detail reachable by `--help`.
The host has no shape at all — it has a flat array of strings in
`apps/server/src/agentGateway/harnessPolicy.ts` that describes a command surface
it is not structurally connected to. Nothing keeps the two in agreement because
they are not the same kind of artifact.

---

## Evidence

Everything below was verified against the working tree or by running the command.
Line numbers are from 2026-08-22.

### The discovery conditional

`apps/server/src/agentGateway/harnessPolicy.ts` renders, among 21 bullets:

> Start Penkra work with `penkra --help` when the relevant hierarchy is unknown.

and separately:

> Penkra Apps are locally installed visual applications scoped to a Space. Use
> `penkra apps list` to establish which Apps are actually enabled in the caller
> Thread's Space; then use `<app-slug> --help`. Never infer that an App is
> installed, enabled, or capable from the user's request, a Skill, a native
> application, source files, prior knowledge, or a similarly named provider
> capability.

The second sentence names a command whose payload is a subset of the first
command's payload. The final sentence is a prohibition with no procedure attached:
it says what not to conclude, never what to do instead.

### `penkra --help` already carries the catalog

`apps/server/src/appRuntimeCli.ts:520`:

```ts
function coreHelp(catalog, additionalCoreCommands): unknown {
  return {
    description:
      "Penkra registered commands run through penkra_exec_command; they are not shell commands.",
    commands: [
      ...additionalCoreCommands,
      ...APP_DEVELOPER_COMMANDS,
      "penkra apps list",
      "penkra tabs current" /* ... */,
    ],
    appCommands: summarizeCatalog(catalog).map((app) => ({
      root: app.slug,
      help: `penkra_exec_command: ${app.slug} --help`,
      operations: app.operations,
    })),
  };
}
```

Live output on this machine returns six Apps — `apps`, `borge`, `borge-studio`,
`browser`, `canvas`, `explorer` — with every operation each declares.

### Provider reality versus provider declaration

`packages/shared/src/providerMetadata.ts` declares all nine `ProviderKind`s with
`available: true`. `penkra capabilities` at runtime reports something else:

```
codex        enabled  available    7 models   source: managed-connections
claudeAgent  enabled  available    4 models   source: managed-connections
opencode     enabled  available   29 models   source: managed-connections
cursor       enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
antigravity  enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
grok         enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
droid        enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
kilo         enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
pi           enabled  UNAVAILABLE  0 models   "Provider runtime is not installed."
```

Two different fields named `available` mean two different things: "Penkra has
written an adapter" and "the provider CLI is installed and usable." The static one
is misleading enough that it produced a wrong conclusion during this audit.

### One policy, five delivery sites, four mechanisms

| Site                                     | Mechanism                                  | Note                     |
| ---------------------------------------- | ------------------------------------------ | ------------------------ |
| `provider/Layers/ClaudeAdapter.ts:970`   | `systemPrompt.append`                      | computed capability flag |
| `provider/Layers/ClaudeAdapter.ts:4661`  | MCP server `instructions`                  | **hardcodes `true`**     |
| `agentGateway/Layers/AgentGateway.ts:86` | `AGENT_GATEWAY_INSTRUCTIONS`               |                          |
| `codexAppServerManager.ts:447`           | concatenated after `</collaboration_mode>` |                          |
| Cursor / Grok / Droid / OpenCode / Pi    | per-session text part                      | two different helpers    |

Claude receives the policy twice. When the gateway is degraded, the two copies
disagree about Claude's own capabilities, because line 4661 asserts full control
unconditionally while line 970 renders the degraded variant.

This counts the tree as it stands today, including the last row's five adapters.
Decision 14 deletes four of them — Cursor, Grok, Droid, and Pi — leaving Codex,
Claude, and OpenCode. Part 3.X is the same evidence recounted against that
narrower set, which is why it says _three_ mechanisms where this table says four.
Neither number is wrong; they are before and after.

### Provider-specific prompt text has no governing rule

`ClaudeAdapter.ts:962` appends six lines. Classified:

| Line                                                                                  | Truly provider-specific?                                |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| "You are running inside Penkra, a coding app that embeds the Claude Agent SDK."       | No — duplicates the policy's own identity line          |
| "Do not present the host app as Claude Code unless the user is explicitly asking."    | No — fights a preset Penkra opted into                  |
| "Treat the current working directory as the active workspace."                        | No — universal                                          |
| "When the user asks about the current project, proactively inspect files."            | No — universal                                          |
| "When spawning subagents, set the Agent tool's `model` parameter, worker-`<tier>`."   | No — duplicates the agent definitions' own descriptions |
| "Honor explicit user instructions about model or effort; otherwise match complexity." | No — Penkra policy, provider-neutral                    |

`DroidAdapter.ts:153` adds a single unrelated line about serializing CPU-heavy
work. There is no rule determining what may live in an adapter, so text landed
wherever someone was editing.

### The preset is opt-in

Per the Claude Agent SDK documentation, omitting `systemPrompt` yields a minimal
prompt containing only essential tool instructions; Claude Code's persona and
guidelines arrive only if `preset: "claude_code"` is requested. Penkra requests it
at `ClaudeAdapter.ts:4708` and then spends a line denying it.

### The command surface re-parses structure it was given

`apps/server/src/agentGateway/hostToolContract.ts`:

```ts
export const PENKRA_EXEC_COMMAND_ZOD_SHAPE = {
  command: z.string().describe('One registered command, for example: "penkra --help" ...'),
};
```

`apps/server/src/appRuntimeCli.ts:551`:

```ts
export function tokenizeRegisteredCommand(command: string): string[] {
  if (typeof command !== "string" || !command.trim()) return [];
  if (/[$`]/.test(command)) {
    throw new Error("Command expansion is not supported by penkra_exec_command.");
  }
  // ... quote state machine, backslash escapes, operator rejection ...
}
```

The `$` and backtick check runs on the raw string before tokenization, so it fires
inside quotes and defeats the escape handling defined twelve lines below it. There
is no way to send a literal `$`.

Observed consequences in one session:

- Canvas variable references (`"fill": "$fog"`) are unsendable. Canvas's own schema
  uses `$` for variables, so theme-aware fills cannot be written at all.
- Ordinary content is unsendable: `"$49.99"`, or prose containing a backtick.
- `--input` with escaped quotes failed JSON validation because the payload is
  JSON encoded into a string encoded into JSON.
- The only working escape was `String.fromCharCode(36) + "fog"`, which no agent
  will discover.

### Definitions

`grep` across `docs/`, `README.md`, and `AGENTS.md` finds no definition of Space,
Thread, Project, operation, controller, installation, or tab. `docs/app-development.md`
uses "Space" at lines 101, 102, 103, 109, 111, 177, 181, 182, 184, 185, 200, 204,
205, 222, 287, 305, 444, 452, 453, 474, 481, 483, 532 and elsewhere.

Line 204 reads: "A Space ID is context an App may use, not a claim that App data is
automatically Space-owned or shared with Space members." This corrects a
misreading of a term the document never introduced, and implies multi-user
semantics that nothing else explains.

### Skills

Implemented and load-bearing:

- `apps/server/src/appSkillsCatalog.ts` — loads enabled, Space-scoped Skills from
  verified immutable App packages; throws if `scope !== "app:" + slug`.
- `packages/sdk/src/manifest.ts:82` — `AppSkillDeclaration`, "Package-relative
  directory containing one Agent Skills-compatible SKILL.md."
- `packages/contracts/src/providerDiscovery.ts` — `ProviderSkillDescriptor`,
  cross-provider catalog, explicit Space indexing.
- `codexAppServerManager.ts:830` — registers `~/.penkra/skills` as a Codex skill
  root.

Documented for App authors in one sentence, `docs/app-development.md:95`:
"Settings and Skills are declarative contributions interpreted by the host. See
the exported TypeScript declarations in `@penkra/sdk` for the authoritative field
types and validators."

The injected policy, meanwhile, tells agents what a Skill _is not_ — "A Skill
supplies instructions, never capabilities" — without ever saying what one is or
that Apps ship them.

### Repository residue

```
.penkra-voice-session-qa/            751M
.penkra-ui-overflow-check/           482M
.penkra-canvas-sideload-live-root/   310M   (plus -v2, another 310M)
.penkra-sideload-qa-bundle/          259M
six further dirs at                  173M each, byte-identical Electron trees
-------------------------------------------
.penkra-* total                      3.3G
release-local/                        311M
.tmp/                                  10M
repo total                             27G
```

Sixteen scratch roots. Four contain a nested `codex-home-overlay/AGENTS.md`. All
are gitignored at `.gitignore:19` and untracked, so history is clean and the
problem is purely that nothing deletes them.

The parent workspace directory additionally holds artifacts from unrelated work:
`admin-schoolbaseapp-com-titan-2026-07-31.tar.gz`,
`ceo-studentsindemand-com-titan-2026-07-31.tar.gz`, `export-titan-mailbox.pl`,
`finish-megachapel-email-migration.command`, and two dated desktop staging
directories.

---

## Decisions already made

These were settled in discussion and are not reopened below.

1. Drop `systemPrompt: { preset: "claude_code" }`. Penkra supplies its own system
   prompt.
2. Provider adapters carry zero prompt prose. Delete
   `buildEmbeddedClaudeSystemPromptAppend` and `DROID_RESOURCE_DISCIPLINE_PROMPT`.
   Adapters decide delivery mechanism only.
3. `penkra` becomes App zero: an `INSTRUCTIONS.md` plus a declared operation set,
   assembled by the same builder that assembles `<slug> --help`.
4. Penkra's document is injected at session start, not fetched. Root `penkra --help`
   returns the identical document so the two can never diverge.
5. Discovery is unconditional. No instruction may key on the agent's own belief
   about whether it needs to look.
6. Remove `PENKRA_HARNESS_POLICY_VERSION` from rendered text.
7. Replace `command: string` with a structured argv array plus a structured
   `input`. Delete the tokenizer's guards rather than adjusting them.
8. Canvas's five `guidelines.get` topics inline into Canvas's `INSTRUCTIONS.md`.
9. Skills split by audience: authoring contract in `docs/app-development.md`,
   usage and trust semantics in Penkra's `INSTRUCTIONS.md`. No overlap.
10. Delete the scratch roots. The `.gitignore` rule remains a safety net, not the
    policy.
11. Prohibitions are not a section. Where a hazard is real, explain it in place,
    with its reason, at the point where an agent would hit it.
12. `docs/app-development.md:204`'s "shared with Space members" is wrong. No
    membership model exists. Remove the phrase.
13. A failed gateway registration surfaces a session error. Delete the degraded
    two-bullet policy variant, `gatewayControlAvailable`, and
    `PROVIDERS_WITH_THREAD_SCOPED_PENKRA_MCP`.
14. Remove `cursor`, `antigravity`, `grok`, `droid`, `kilo`, and `pi` from
    `ProviderKind`. `ProviderKind` becomes `codex | claudeAgent | opencode`.
15. The structured-command change is a hard cut. No dual-shape release.
16. All affected Apps are updated in the same pass as the platform change, so
    breaking an App's published operation set is acceptable when the App ships
    with it. `canvas guidelines.get` is removed, not deprecated.
17. `AGENTS.md` is canonical. `CLAUDE.md` becomes a pointer to it.
18. Remove the legacy Home chat container and the invariants that special-case it.
19. **SUPERSEDED by 22**, which picks the winning word rather than only committing
    to pick one. Kept for the reasoning it records.
    Resolve the folder/project vocabulary split. The UI says folder; the contracts
    say project. One word wins and the other is migrated.
20. Remove managed chat containers entirely. There are no loose chats. Every thread
    lives in a folder and every folder lives in a Space, with no exception for a
    system-owned container. `ContainerKind` disappears with them.
21. **PARKED** — `space.reorder`, `sidebar.item.move`, and folder movement in
    general are out of scope for this pass on the owner's instruction. The findings
    stay recorded (9.8, 9.9) and the decisions stay unmade.
22. **Folder** is the word, everywhere. The contracts migrate to it, not the UI away
    from it, and the agent-facing command becomes `penkra folders list`.
23. **PARKED** with 21. The three-path divergence is real and recorded in 9.8; the
    fix is deferred, not rejected. The one part that is _not_ parked is stopping
    `folder.update` from accepting `spaceId` at all, since that is what makes a
    metadata command a movement command.
24. Drop `.meta` everywhere — `space.meta.update`, `project.meta.update`, and
    `thread.meta.update`. A command is named for the entity whose state it changes.
    A nested segment is earned only by a child that has no identity outside its
    parent; `meta` is not a child. (The `space.projects.assign` half of this rule
    is parked with 21 and 23; the naming argument stands and is recorded in 11.4.)
25. Prune thread markers. `thread.marker.add` / `.remove` / `.done.set` /
    `.label.set`, `ThreadMarkerId`, and the projection rows go.
26. No agent-facing operation takes a `spaceId`. The caller's Space is derived from
    its thread via its folder; `requireThreadSpaceId` already does this.
27. Delete `penkra threads create-many`. One `create`, called as many times as
    needed.
28. Accept partial creation (A7, option 1). The compensating saga goes with the
    tool. A failure on call 3 leaves threads 1 and 2, and the error must say so.
29. The earned-segment rule is **advisory**. `docs/app-development.md` gives
    recommended examples and the reasoning; `penkra app test` does not enforce it.
30. Guard `penkra threads send` against self-targeting (B4 / 12.7), and state both
    failure modes in the text rather than relying on the agent having read a rule.
31. Loose threads migrate into a per-Space `Chats` folder, using the Space already
    recorded on each thread. No new product decision is needed; the data is there.

## Open questions

None outstanding. The last two were closed on 2026-08-22:

- **`spaceId` nullability** — resolved by decision 20. Managed chat containers are
  being removed entirely, so `kind: "chat"` disappears and `spaceId` becomes
  non-nullable with no exception to carve out.
- **Folder or project** — resolved by decision 22. **Folder** wins everywhere,
  including the agent-facing command, which becomes `penkra folders list`.

---

## Discussion inventory

Everything that still needs a decision, an argument, or a draft. Maintained as the
single answer to "what is left" so no part of it lives only in conversation.

### A. Decisions the owner has made (2026-08-22)

| #   | Question                              | Answer                                                                          |
| --- | ------------------------------------- | ------------------------------------------------------------------------------- |
| A1  | Collapse `create` / `create-many`?    | **Delete `create-many`.** One `create`, called N times.                         |
| A2  | May `folder.update` accept `spaceId`? | **No.** No agent-facing op takes a `spaceId`; it is derived from the caller.    |
| A3  | Does `penkra --help` survive?         | **Keep the command, stop mentioning it.** Instructions are loaded, not fetched. |
| A4  | App catalog at session start?         | **Yes**, rendered in full.                                                      |
| A5  | Skills own section?                   | **Yes.**                                                                        |
| A6  | Keep thread markers?                  | **No — prune the feature.**                                                     |

### A-open. Decisions still owed

**None.** A7 and A8 were answered on 2026-08-22 (decisions 27, 28 and 29); A9 was
withdrawn as already-sequenced; A10 was not a decision but an audit, now done
(Part 3.X).

### B. Arguments now made (were assertions)

- **B1 / B2** — still owed as prose; the finding is established, the writing is not.
- **B3** — still owed; blocked on audit D2.
- **B4 — ANSWERED, 2026-08-22.** See Part 12.7. `penkra_send_message` writes a
  message with `role: "user"` and starts a new turn. Used on the caller's own
  thread it fabricates a user message in the transcript and stacks a turn on the
  live one. There is no self-target guard.

### C. Drafts owed

| Part | Owed                                                                                         | Size                                  |
| ---- | -------------------------------------------------------------------------------------------- | ------------------------------------- |
| 1    | Eight definitions: Thread, Folder, Operation, Controller, Tab, Installation, Skill, Sideload | ~200 lines                            |
| 2    | Penkra's own `INSTRUCTIONS.md`, complete                                                     | ~250 lines                            |
| 5    | Four Canvas sections; only "Before you write anything" exists                                | ~300 lines                            |
| 6    | The Skills authoring section                                                                 | ~80 lines                             |
| 7    | `docs/app-development.md` rewrite; currently a checklist                                     | requires reading all ~540 lines first |
| 12   | Nothing yet drafted; 12.1–12.7 are findings, not replacement text                            | ~150 lines                            |

### D. Audits not yet performed

- **D1.** Part 10.3 — the Thread-orchestration policy bullets. Gates Part 2.
- **D2.** Part 10.4 — the untrusted-data boundary. Gates Part 2.
- **D3.** Part 12.6 — the eight read and diagnostic tools.
- **D4.** The `App` manifest surface end to end; Part 7 has only sampled it.
- ~~**D5.** Provider adapters other than Claude and OpenCode.~~ **Done**
  (2026-08-22). Codex was read; the result is Part 3.X. It did not confirm the
  earlier picture — it added a third delivery mechanism and a third
  `gatewayControlAvailable` value, so Part 3's checklist changed rather than
  gaining a footnote. Two adapters that decision 14 deletes were not read, and do
  not need to be.
- **D6.** Every error string reachable from an agent-invoked operation (9.7 scopes
  this; nobody has run it).

### E. Unsolved technical problems

- **E1.** Canvas document corruption (9.1). Root cause unknown. Ruled out: nested
  inserts, `Object.assign`, `padding:[0,16]`, `cornerRadius:9999`,
  `justifyContent:"space_between"`, dangling refs. Open: duplicate node IDs, a
  single-Insert size limit, `fontWeight` as string versus number.
- **E2.** Two Canvas documents are unreadable and undeletable _right now_
  (9.1 + 9.2). This is live data loss, not a plan item.

### F. Parked by owner instruction

Recorded so nothing is silently dropped. These findings are real and the fixes are
deferred, not rejected.

- **F1.** Folder movement, reordering, and `sidebar.item.move` in general —
  decisions 21 and 23, findings 9.8 and 9.9.
- **F2.** `space.reorder` removal.

---

## Part 0 — The writing standard

Everything below is written to one standard. It is stated here once so the rest of
the plan can refer to it.

**Write for a competent newcomer, not for a rule engine.** The reader is capable
and has never seen this system. They will do the right thing if they understand
the situation, and no volume of prohibition substitutes for that understanding.
Anthropic's guidance on tool authoring puts it as describing the tool "to a new
hire on your team," and notes that small refinements to descriptions produce
disproportionate improvements in agent behaviour.

**Every rule carries its reason.** "Never infer that an App is installed" tells a
reader they were about to be wrong without telling them why anyone would infer, or
what to do instead. A reason converts a rule into a judgement the reader can
extend to cases the rule never anticipated. Rules without reasons are followed
literally and abandoned under pressure.

**Procedure beats prohibition.** For every hazard, the document should answer:
when does this come up, what do I do, how do I tell if it worked, what does
failure look like. A hazard with no procedure attached is a warning label on a
door with no handle.

**Say what a thing is before saying what it is not.** The current policy defines
App in terms of Space without defining Space, and defines Skill entirely by
negation. Both are unreadable to someone who does not already know.

**Hedge where the world is uncertain.** An App declaring `issues.create` is
_probably_ an issue tracker. Writing "is" teaches agents to conclude from partial
evidence, which is the same failure the discovery rules exist to prevent. The
document must model the epistemics it is asking for.

**Depth where it is earned, brevity where it is not.** The MCP server-instructions
guidance names "don't write a manual" and excessive length as anti-patterns, and
that is correct for always-injected surfaces. It is not an argument for thin
documents behind an explicit fetch. Penkra's injected document stays tight and
procedural. An App's `INSTRUCTIONS.md`, reached only by an agent that has already
decided to use that App, is as thorough as the subject requires.

**No dead prose.** If a sentence does not change what a reader does, delete it.
Version markers, restatements of tool descriptions, and reassurances about
Penkra's qualities all fail this test.

References:

- https://www.anthropic.com/engineering/writing-tools-for-agents
- https://blog.modelcontextprotocol.io/posts/2025-11-03-using-server-instructions/
- https://docs.claude.com/en/docs/agent-sdk/modifying-system-prompts

---

## Part 1 — Definitions

**Why first.** Every other document is written on top of these words. Rewriting
prose before the nouns are fixed produces fluent text resting on the same
ambiguity.

**Where they live.** A new `docs/concepts.md`, which is the single normative
source. Penkra's `INSTRUCTIONS.md` carries short operational restatements — one or
two sentences, enough to act on. `docs/app-development.md` links rather than
restates. Any term defined in two places will drift; the rule is one definition,
many links.

**Terms requiring definition**, each with: what it is, what contains it, what it
contains, what it is commonly confused with, and what an agent can do with it.

| Term         | Status today                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| Space        | Used 30+ times in the public App contract, never defined. Derived below.                                                        |
| Thread       | Central to the command surface, never defined.                                                                                  |
| Folder       | Appears in `penkra projects list`, never defined, and named `project` in every contract (Part 11.3 settles this on **folder**). |
| App          | Defined only as "visual applications scoped to a Space" — in terms of Space.                                                    |
| Operation    | Used constantly; the dotted-key/word-form duality is explained, the concept is not.                                             |
| Controller   | Appears at `docs/app-development.md:287` with no introduction.                                                                  |
| Tab          | Conflated with browser tabs throughout.                                                                                         |
| Installation | Distinct from App and from enablement; never distinguished.                                                                     |
| Skill        | Defined only by negation.                                                                                                       |
| Sideload     | Used as a verb in the developer commands with no definition.                                                                    |

**Draft — App.** Written to show the target register; Space is left as a slot
until the open question is answered.

> An App is a program you install into Penkra that has its own window, its own
> storage, and its own set of operations an agent can call. Apps come from the
> Penkra registry or are sideloaded during development.
>
> Two things about an App matter to an agent. It has a **slug** — a short unique
> name like `canvas` or `browser` that is the first word of every command that
> App accepts. And it declares **operations** — named actions with validated
> inputs, like `documents.create` or `pages.navigate`, which an agent invokes as
> `canvas documents create`.
>
> An App's window and its operations are the same program but not the same
> surface. Opening Canvas's window does not let you call its operations, and
> calling `canvas documents create` does not open a window. Some work needs both.
>
> Apps are isolated from each other. One App cannot read another's storage, call
> another's operations on its behalf, or see another's tabs.

That is roughly 180 words, replacing a fifteen-word fragment that referenced an
undefined term. It is longer because it is usable.

### Space, derived from the schema

Not blocked. `OrchestrationSpace` at `packages/contracts/src/orchestration.ts:437`
settles it:

```ts
export const OrchestrationSpace = Schema.Struct({
  id: SpaceId,
  name: SpaceName, // trimmed, non-empty, max length
  icon: SpaceIconName, // one of a fixed set: "target", "tree", "school",
  // "backpack", "gamecontroller", "camera-1", ...
  sortOrder: NonNegativeInt,
  createdAt: IsoDateTime,
  updatedAt: IsoDateTime,
  archivedAt: IsoDateTime | null,
  deletedAt: IsoDateTime | null,
});
```

Supporting facts:

- The command set today is `space.create`, `space.meta.update`, `space.reorder`,
  `space.archive`, `space.restore`, `space.delete`, and `space.projects.assign`.
  After decisions 21, 23 and 24 it is `space.create`, `space.update` (name, icon,
  position), and `space.archive` / `.restore` / `.delete`. Membership leaves the
  Space namespace entirely and becomes `folder.move`, because the folder is the row
  that changes. None of these is agent-facing: they run on the internal shell command bus, not
  through `penkra_exec_command`. They are described here because the definitions in
  this Part have to match the model they implement. See Part 11.4.
- **A command is named for the entity whose state changes, and a nested segment is
  earned only by a child with no identity outside its parent.** Moving a folder
  changes the folder, so it is `folder.move`, not `space.projects.assign`.
  `thread.marker.*` and `thread.message.*` earn their segments — a marker or a
  message does not exist apart from its thread. `meta` earns nothing: there is no
  meta entity, only an adjective for "fields someone judged boring," and that
  judgement is exactly what let `spaceId` and `kind` into a metadata command.
  Part 11.4 carries the rule, the full table, and the worked example.
- **Every folder belongs to exactly one Space, and every Thread belongs to exactly
  one folder. Nothing floats loose.** The schema types `spaceId` as
  `optional(NullOr(SpaceId))`, but that nullability is not a product option; it
  exists only for two system containers, and both are being removed. Managed chat
  containers (Part 11.2) are excluded by `kind`, and legacy Home rows (Part 11.1)
  kept `kind: "project"` and are recognised by title string plus workspace path —
  `apps/server/src/orchestration/commandInvariants.ts:175` calls them "reachable
  from every Space, so they must never belong to one," and the decider blocks
  renaming them so that fragile signal cannot drift. Once both are gone `spaceId`
  is non-nullable and this bullet is simply true, with no exception attached.
  **This nullability is what produced a wrong Space definition during this audit**,
  which is the argument for closing it rather than documenting around it.
- App installation state is keyed per Space and per App:
  `spaceStateByKey["personal\0com.acme.figma"]`, carrying `enabled`,
  `permissions`, and settings. Enablement is per-Space.
- The default is `"personal"`.
- **There is no membership model.** No `spaceMember`, no sharing command, no
  member field anywhere in the contracts. A Space is local to this installation.

So:

> A Space is a workspace you create in Penkra to keep one area of your work
> separate from another — it has a name, an icon, and a position in the sidebar.
> You might have one for a job and one for personal projects.
>
> Everything lives inside one: a folder belongs to exactly one Space, and a
> conversation belongs to exactly one folder. Nothing floats loose.
>
> A Space also decides which Apps are on. The same App can be enabled in one
> Space and off in another, with its own permissions and settings in each. That
> is why the catalog above is specific to this Space rather than to your account:
> an App you have used before may simply not be enabled here.
>
> Spaces are local to this installation and are not shared with anyone.

The last line contradicts `docs/app-development.md:204`, which says a Space ID is
"not a claim that App data is automatically Space-owned or shared with Space
members." Nothing in the schema supports members. Flagged as an open question.

**Deliverables**

- [ ] Write `docs/concepts.md` covering all ten terms to the standard above.
- [ ] Reconcile the "Space members" language in `docs/app-development.md:204`.
- [ ] Audit `docs/app-development.md` for first uses of each term; link, do not restate.
- [ ] Audit `docs/app-development-internals.md` for the same.
- [ ] Grep for terms used before their link and fix ordering.

---

## Part 2 — Penkra as App zero

### The shape

Today the host is a string array; Apps are documents plus operations. Make them
the same kind of thing.

```
penkra-apps/<app>/
  INSTRUCTIONS.md          returned by `<slug> --help`, with the operation list
  penkra-app.json          slug, name, summary, operations[] with per-op summary

apps/server/src/agentGateway/instructions/
  INSTRUCTIONS.md          injected at session start; identical from `penkra --help`
  operations.ts            the core operation set, declared not hand-listed
```

One builder assembles both. `assembleInstructions(doc, operations, catalog?)`
returns the document with the operation list rendered from declarations, so the
prose can no longer disagree with what exists — the current `penkra apps list`
redundancy becomes structurally unrepresentable.

### Delivery

Injected once per session. The existing `takePenkraHarnessPolicyForSession` latch
is the right mechanism and survives; what changes is the content and the number of
call sites.

| Provider                                    | Channel                                                           |
| ------------------------------------------- | ----------------------------------------------------------------- |
| claudeAgent                                 | MCP server `instructions` — **once**, not also via `systemPrompt` |
| codex                                       | MCP server `instructions`                                         |
| opencode                                    | MCP server `instructions`                                         |
| any provider not honouring MCP instructions | session text part, same content                                   |

Adapters choose the channel. They do not author, edit, extend, or duplicate the
content. Delete `AGENT_GATEWAY_INSTRUCTIONS` as a separate constant, the
`</collaboration_mode>` concatenation at `codexAppServerManager.ts:447`, and the
second Claude delivery.

**Why injection rather than fetch.** An agent can discover Canvas from inside
Penkra. It cannot discover Penkra from inside Penkra. Penkra is the ground the
discovery procedure stands on, so it must arrive before the first turn. The
document's _shape_ stays identical to every App, which is the consistency that
matters; only its delivery differs, and only because it has to.

**Why root `--help` returns the same bytes.** Two renderings of the same subject
drift. Making them one artifact costs nothing and removes a class of bug. Nested
help — `penkra threads create-many --help` — remains a genuine probe, because
per-operation input contracts are too large to inject and are only needed on
demand.

### The catalog

Rendered from live state at injection time, using the `summary` field already
required of every App manifest. No new manifest field is needed.

```markdown
## What is installed right now

Apps enabled in this Space, with the summary each App's author wrote:

apps Install, update, and manage Penkra Apps.
listings.open · installations.install · installations.update ·
installations.uninstall · installations.enable ·
installations.disable · installations.remove-data

borge Universal find-anything research assistant.
load.skill · compute · web.read · forage.request · x.search ·
linkedin.people.search

browser Browse and evaluate pages in a hosted browser surface.
pages.open · pages.navigate · pages.evaluate

canvas Design documents on an infinite canvas.
guidelines.get · documents.list · documents.get ·
documents.create · documents.open · documents.mutate ·
documents.execute · documents.export · sharing.list ·
sharing.add · sharing.remove · selection.set ·
viewport.focus · performance.snapshot

Summaries and operation names are written by each App's author. They are a
starting point for investigation, not a specification. Run `<slug> --help` before
using an App for the first time: it returns that App's full instructions and the
validated input contract for every operation it declares.
```

The catalog exists so that discovery cannot be skipped, not so that it can be
avoided. Depth still comes from probing.

### Draft — the discovery section

This replaces four bullets and one paragraph of prohibitions.

> ## Finding the right capability
>
> Penkra Apps are chosen and installed by the user, so what is available here is
> not something you can predict. An App's name tells you little: names are picked
> by their authors, two Apps can do the same kind of work, and an App with a
> familiar name may be unrelated to the thing you are thinking of. The catalog
> above is the ground truth for this session. Read it before you decide what a
> request is about.
>
> Look again — do not rely on memory of the catalog — whenever:
>
> - the request names a capability: design, browse, search, file, schedule, draw,
>   edit, publish, track, review;
> - the request names something you do not recognise, especially a proper noun;
> - the request points at something on screen: "this", "the current one", "that
>   document", "here";
> - you are about to use one of your own tools for work a visual App might own.
>
> That last case is the one that goes wrong most often, and it goes wrong quietly.
> You have tools of your own, and they are usually the right instinct. But when a
> user asks for something a Penkra App exists to do, doing it with your own tools
> produces work in the wrong place — a file the App cannot see, a document that
> does not appear in the user's account, a browser session the user cannot watch.
> The output looks correct and is useless. Check the catalog first; it costs one
> command.
>
> **When several Apps could fit** — two browsers, two design tools — resolve in
> this order:
>
> 1. **A visible tab wins.** `penkra tabs current` tells you what the user is
>    looking at. If a Canvas tab is open and the user says "make the header
>    bigger," they mean that document in that App. What is on screen is usually
>    better evidence than the words, because the user is describing what they see.
> 2. **An App already used this session wins** over one that has not been.
> 3. **Otherwise, ask.** Name the candidates and say what distinguishes them.
>    Choosing between two equally plausible Apps is a coin flip performed on
>    someone else's data.
>
> **What is not evidence that an App is available here:** your own tool list, the
> text of a Skill, an application installed on this machine, a file in the
> repository, a service you know from training, or the way the user phrased the
> request. These are the sources that feel most like knowledge and are most often
> wrong, because each describes a different system that happens to share
> vocabulary with this one. A tool of yours called `pencil` and an App called
> `canvas` may both edit designs; they are unrelated, store data in different
> places, and work done in one does not appear in the other.
>
> **Reading the catalog.** Read operations, not names. An App named `atlas`
> declaring `issues.create`, `issues.list`, and `issues.close` is probably an
> issue tracker — but operation names are authored too, and can mislead as easily
> as slugs. Before you write, delete, send, or do anything at all, run
> `<slug> --help` and read the operation's own description and input contract.
> The catalog tells you what to investigate. It does not tell you what an App is.

### Section order of the injected document

1. What Penkra is, and what you are within it
2. The command surface — grammar, structured input, what it is not
3. What is installed right now — the rendered catalog
4. Finding the right capability — the discovery section above
5. Reading the screen — tabs, ambient state, what a snapshot is and is not
6. Skills — what they are, how far to trust them
7. Threads and Projects — what they are, when to create them
8. When things fail — the failure-mode section, Part 9

Target 900-1,300 words. The current policy is roughly 700 words of denser, less
usable text; the growth is reasons and procedures, not padding.

### Deliverables

- [ ] `assembleInstructions()` builder shared by host and Apps.
- [ ] Declare core operations as data; delete the hand-maintained `commands` array
      at `appRuntimeCli.ts:520`.
- [ ] Write `INSTRUCTIONS.md` to the section order above.
- [ ] Render the catalog with `summary` from each manifest.
- [ ] Root `penkra --help` returns the injected document verbatim.
- [ ] Delete `harnessPolicy.ts`: the bullet array, `PENKRA_HARNESS_POLICY_VERSION`,
      `PENKRA_HARNESS_POLICY_MARKER`.
- [ ] Collapse five delivery sites to one per provider.
- [ ] Test: injected bytes equal `penkra --help` bytes.
- [ ] Test: catalog matches installed Apps for a fixture Space.
- [ ] Test: no App or operation is named in prose without appearing in the catalog.

---

## Part 3 — Provider layer

### Drop the preset

`ClaudeAdapter.ts:4708` currently requests Claude Code's full system prompt and
then appends a line instructing the model not to present as Claude Code. Both
halves go.

```ts
// before
systemPrompt: {
  type: "preset",
  preset: "claude_code",
  append: buildEmbeddedClaudeSystemPromptAppend(agentGatewayCredentials !== undefined),
}

// after
systemPrompt: PENKRA_SYSTEM_PROMPT,   // provider-neutral, one source
```

Consequences to work through rather than assume:

- Claude Code's preset carries coding conventions, response-style guidance, and
  project-context behaviour that Penkra has been inheriting for free. Dropping it
  means Penkra must state what it actually wants. This is the point — inherited
  behaviour Penkra never chose is behaviour Penkra cannot reason about — but it is
  real work, not a deletion.
- `settingSources` and `CLAUDE.md` loading are separate mechanisms and must be
  checked independently; dropping the preset should not silently change whether
  project instructions load.
- Behavioural comparison before and after is required on a real task set. "It
  still works" is not a finding; the failure mode is subtle degradation in code
  quality and tone, not breakage.

### Delete adapter prose

```ts
// DELETE — ClaudeAdapter.ts:962
export const buildEmbeddedClaudeSystemPromptAppend = (gatewayControlAvailable: boolean) => [ ... ];

// DELETE — DroidAdapter.ts:153
const DROID_RESOURCE_DISCIPLINE_PROMPT = "Keep CPU-intensive validation work serial: ...";
```

Disposition of every line:

| Line                                                                            | Goes to                                                            |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| "You are running inside Penkra, a coding app that embeds the Claude Agent SDK." | deleted — Penkra's document says this once                         |
| "Do not present the host app as Claude Code..."                                 | deleted — no borrowed identity to correct                          |
| "Treat the current working directory as the active workspace."                  | Penkra `INSTRUCTIONS.md`, all providers                            |
| "When the user asks about the current project, proactively inspect files."      | Penkra `INSTRUCTIONS.md`, all providers                            |
| "When spawning subagents, set the Agent tool's `model` parameter..."            | the agent definitions' own descriptions in `.claude/agents/*.md`   |
| "Honor explicit user instructions...otherwise match task complexity."           | same — it is guidance about choosing among those agents            |
| Droid CPU serialisation                                                         | Penkra `INSTRUCTIONS.md` if it is Penkra policy; otherwise deleted |

The subagent lines are the case worth stating explicitly, because they look
provider-specific and are not. The SDK already surfaces each agent type with the
description its definition file carries — "worker-high: General-purpose worker at
high reasoning effort; choose per task complexity" arrives without Penkra doing
anything. Restating it in the system prompt creates a second source that can drift
from the first. If Penkra wants a different policy, it edits the definitions.

**The governing rule, to be written into `penkra/AGENTS.md`:** if a line would be
true for a provider Penkra has not integrated yet, it is host policy and belongs
in `INSTRUCTIONS.md`. An adapter may contain only what is false for every other
provider. Under this rule an adapter's correct prose content is currently zero
lines, and that should be asserted by a test.

### Fix the `available` collision

`packages/shared/src/providerMetadata.ts` declares `available: true` for all nine
providers, meaning "an adapter exists." `penkra capabilities` reports `available`
meaning "the runtime is installed and usable." Same name, different claims, and
the static one is the misleading one.

- [ ] Rename `ProviderDescriptor.available` to `adapterImplemented`.
- [ ] Audit every read. `apps/web/src/session-logic.ts:47` maps it into session
      state and must be checked for which meaning it needs.
- [ ] Remove six providers from `ProviderKind` (decision 14). `ProviderKind`
      becomes `codex | claudeAgent | opencode`. This deletes six adapters, their
      model catalogs, their icons, their discovery paths, and every switch arm
      that handles them. Enumerate before starting; it is a wide change.
- [ ] Check for persisted rows referencing removed providers and decide the
      migration for a user who has one.

### Degraded gateway — decided: surface the error

Two distinct conditions are currently collapsed into one boolean:

**(a) Provider unsupported.** `PROVIDERS_WITH_THREAD_SCOPED_PENKRA_MCP` lists eight
of nine `ProviderKind`s; `antigravity` is excluded. Antigravity is not installed on
any machine checked, so this branch is dormant in practice.

**(b) Runtime registration failure.** Live and reachable:

```ts
// OpenCodeAdapter.ts:3431 — connection status from client.mcp.add(...)
// PiAdapter.ts:2137       — const gatewayControlAvailable = gatewayTools.length > 0;
// ClaudeAdapter.ts:4708   — agentGatewayCredentials !== undefined
```

OpenCode is one of the three live providers, so (b) is not theoretical.

**Why OpenCode can fail where the others cannot.** The three live providers use
three different MCP mechanisms, and only one of them delegates the lifecycle:

| Provider    | Mechanism                                | Can registration fail?                  |
| ----------- | ---------------------------------------- | --------------------------------------- |
| claudeAgent | `createSdkMcpServer` — in-process        | No. There is nothing to connect to.     |
| codex       | Penkra launches and owns the process     | Only if Penkra's own startup fails.     |
| opencode    | `client.mcp.add(...)` — OpenCode owns it | Yes. OpenCode may report not-connected. |

`OpenCodeAdapter.ts:3431` asks OpenCode to add Penkra's server to OpenCode's own
config and then inspects `result.data[PENKRA_MCP_SERVER_NAME].status`. Only that
provider hands lifecycle control to the agent runtime, so only that provider has a
connection that can come back not-connected. It is a difference in mechanism, not
in reliability.

**And it already fails loudly.** The same expression maps a non-connected status
to `Effect.fail(new OpenCodeRuntimeError({ operation: "mcp.add", ... }))`. The
newest adapter already does what decision 13 makes universal, which means the
degraded prompt variant is largely unreachable on the one provider that could
reach it.

- [ ] Delete the two-bullet degraded variant.
- [ ] Delete `gatewayControlAvailable` and every computation of it.
- [ ] Delete `PROVIDERS_WITH_THREAD_SCOPED_PENKRA_MCP` and
      `providerHasPenkraGatewayControl`.
- [ ] Add a session-start error naming the provider, the operation, and the
      underlying failure — not "Penkra MCP control is unavailable."
- [ ] Confirm no remaining caller distinguishes a degraded session from a healthy
      one.

### Deliverables

- [ ] Drop `preset: "claude_code"`; author `PENKRA_SYSTEM_PROMPT`.
- [ ] Behavioural comparison on a real task set, before and after.
- [ ] Verify `settingSources` / `CLAUDE.md` loading is unaffected.
- [ ] Delete `buildEmbeddedClaudeSystemPromptAppend` and `DROID_RESOURCE_DISCIPLINE_PROMPT`.
- [ ] Relocate the two universal lines into `INSTRUCTIONS.md`.
- [ ] Move subagent guidance into `.claude/agents/*.md` descriptions.
- [ ] Rename `available` to `adapterImplemented`; audit reads.
- [ ] Resolve the degraded-gateway question.
- [ ] Test asserting no adapter contains prompt prose.

### 3.X Three live providers, three delivery mechanisms, three capability values

Audit D5 is done. Part 3's earlier conclusions rested on Claude and OpenCode; Codex
is the third available provider and changes the picture.

| Provider | How the policy reaches the model                                                                                                                  | `gatewayControlAvailable`                        | Per session? |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------ |
| Codex    | `PENKRA_GATEWAY_HARNESS_POLICY`, a module constant interpolated into `CODEX_DEFAULT_MODE_DEVELOPER_INSTRUCTIONS` (`codexAppServerManager.ts:447`) | `true`, frozen at import (`harnessPolicy.ts:53`) | **no**       |
| Claude   | `renderPenkraHarnessPolicy({ gatewayControlAvailable })` at `ClaudeAdapter.ts:970`, **and** a second copy hardcoded `true` at `:4661`             | computed **and** `true` — both, in one adapter   | partly       |
| OpenCode | `takePenkraHarnessPolicyForProviderSession` (`OpenCodeAdapter.ts:3761`)                                                                           | computed from real MCP status                    | **yes**      |

Three mechanisms for one job. Only OpenCode does what the abstraction was built
for. Codex never calls the delivery guard at all — it takes the pre-rendered
`true` constant, so the flag can never be false for Codex no matter what the
gateway does, and the "once per session" guarantee that
`takePenkraHarnessPolicyForSession` exists to provide is achieved for Codex only
incidentally, by the string being static.

Worse for the writing goal: on Codex the host policy is **glued to a Codex-specific
collaboration-mode prompt**. `CODEX_DEFAULT_MODE_DEVELOPER_INSTRUCTIONS` opens with
a block about `request_user_input` availability and then concatenates the host
policy onto the end of it. So the one document this plan intends to own arrives, on
one of three providers, as the tail of a provider-specific prompt about a tool that
has nothing to do with Penkra. There is also only one mode constant; if a second
collaboration mode is ever added and someone forgets the concatenation, that
provider silently loses the host policy entirely.

This strengthens the existing decision to delete `gatewayControlAvailable` rather
than weakening it: two of three providers cannot produce a false value, and the
third fails loudly instead (`OpenCodeAdapter.ts:3431`). The flag models a state
that one provider can reach and already refuses to run in.

- [ ] One delivery path for all three providers.
- [ ] Separate the host document from provider-specific prompt text; concatenation
      at a call site is not composition.
- [ ] Delete `PENKRA_GATEWAY_HARNESS_POLICY` and
      `PENKRA_IDENTITY_ONLY_HARNESS_POLICY` — pre-rendered constants are how the
      per-session guard got bypassed.
- [ ] Delete `ClaudeAdapter.ts:4661`'s second copy (9.6).

---

## Part 4 — Command surface

### The defect is the signature, not the guard

```ts
// hostToolContract.ts
export const PENKRA_EXEC_COMMAND_ZOD_SHAPE = {
  command: z.string().describe('One registered command, for example: "penkra --help" ...'),
};
```

A caller with structured data flattens it into a shell-like string; the host then
hand-rolls a parser to recover the structure that was just destroyed. That parser
must decide from characters alone what was a quote, an escape, a separator, or
hostile input. Every decision is a guess. The `$`/backtick guard is one guess,
firing before the quote handling that would otherwise make it unnecessary.

Patching the regex fixes one payload. The next one containing a quote, a
backslash, a newline, or a large script hits the next guess. The class is
eliminated only by not re-parsing.

### The replacement

```ts
export const PENKRA_EXEC_COMMAND_ZOD_SHAPE = {
  command: z
    .array(z.string())
    .min(1)
    .describe(
      'The command as discrete words, e.g. ["canvas","documents","mutate"]. ' +
        "This is not a shell string: there is no quoting, no escaping, and no " +
        "substitution. Send each word as its own array element exactly as it " +
        "should be received.",
    ),

  input: z
    .unknown()
    .optional()
    .describe(
      "Structured payload for the operation, matching the input schema shown by " +
        "`<slug> <operation> --help`. Send JSON directly. Do not serialise it into " +
        "a string; the host validates the object you send.",
    ),

  flags: z
    .record(z.string(), z.union([z.string(), z.number(), z.boolean()]))
    .optional()
    .describe('Named options, e.g. { "document-id": "abc" } for --document-id abc.'),

  tabId: z.string().optional(),
};
```

What this deletes rather than patches:

| Deleted                                  | Why unnecessary                                             |
| ---------------------------------------- | ----------------------------------------------------------- |
| ``/[$`]/`` expansion guard               | No string to expand. `$fog` is a value in an array element. |
| `/[\|&;<>()[\]{}]/` operator guard       | No parsing step for an operator to hijack.                  |
| Quote state machine                      | Array elements have exact boundaries.                       |
| Backslash escape handling                | Nothing to escape.                                          |
| "unfinished escape or quote" error       | Unrepresentable.                                            |
| JSON-encoded-into-string double encoding | `input` is JSON, not a string containing JSON.              |

Every failure observed in the Canvas session maps to a row above. They were one
design decision producing four symptoms.

### Migration

`tokenizeRegisteredCommand` remains only if a genuine string-input surface exists
outside the agent tool; that must be established, not assumed. If it does, it
becomes a thin adapter into the structured path, and its guards move to the one
place where a string genuinely must be parsed.

Breaking-change surface, to be enumerated before starting:

- [ ] `hostToolContract.ts` schema, description, and annotations.
- [ ] Every adapter injection path constructing example commands.
- [ ] Every `--help` example string across core and Apps.
- [ ] `appRuntimeCli.ts` dispatch.
- [ ] `INSTRUCTIONS.md` grammar section, written against the new shape from the
      start rather than retrofitted.
- [ ] Tests asserting the old string shape.

Hard cut (decision 15). No dual-shape release, no compatibility window.

Separately, and not solved by this change: the 100,000-byte script ceiling on
`canvas documents execute`. Structured input removes the encoding pain but not the
limit. Whether a file-handle path is needed is a limits question, tracked in
Part 9.

### Deliverables

- [ ] Enumerate every consumer of `tokenizeRegisteredCommand`.
- [ ] New structured schema with descriptions written to the Part 0 standard.
- [ ] Delete the guards; do not relocate them.
- [ ] Update every first-party App's examples in the same pass.
- [ ] Regression tests for `$`, backticks, quotes, newlines, and nested JSON,
      each asserting success rather than a clean error.

---

## Part 5 — App instructions, with Canvas as the first case

### The contract

Every App ships one `INSTRUCTIONS.md`, returned by `<slug> --help` together with
its operation list. One file, sectioned — not several files. The rule is loading
granularity: content loaded together lives together, because separate files that
are always concatenated drift, duplicate definitions, and eventually contradict
each other.

Section order, uniform across Apps:

1. **What this App is** — one paragraph. What it does, what it operates on, where
   its data lives.
2. **Before you write anything** — preconditions. What must be fetched, checked,
   or understood first, and what breaks if it is not.
3. **How to do the common thing** — a worked example, start to finish, with real
   commands and real responses.
4. **Reference** — every operation, its input contract, its output.
5. **When things fail** — symptom, cause, recovery.

Section 2 is the one that does not exist anywhere today, and it is the one that
would have prevented this session's failures.

### Canvas today

`penkra-apps/canvas/INSTRUCTIONS.md` is 37 lines. It says:

> Begin with `canvas documents list` ... Use `canvas guidelines get --topic <topic>`
> for detailed workflow, execute, design, text, and review guidance.

and

> Script errors, timeouts, invalid documents, and conflicts do not commit partial
> edits.

Three problems.

**Lazy-loading behind lazy-loading.** `guidelines.get` defers roughly 1,400 words
across five frozen topics. But `canvas --help` is _already_ on demand — it is only
fetched by an agent that has decided to use Canvas. A second deferral behind the
first buys no tokens on any session that does not touch Canvas, and costs a round
trip plus the real chance the agent skips it, on every session that does. Inline
all five topics. What the split was protecting was ordering, not cost, and
ordering is solved by section order.

**No schema precondition.** Nothing tells an agent that writing to a `.pen`
document requires knowing its node schema, or how to obtain it. In this session
that omission produced two permanently unreadable documents in the user's account.
By contrast the Pencil MCP server states the precondition as a blocking rule —
"Knowing the schema is required to use any other Pencil MCP tool" — and agents
follow it.

**A durability claim contradicted by observation.** A single ~150-node `Insert`
made a previously healthy document permanently unreadable. The document was
verified healthy immediately before — export succeeded, `return 1+1;` succeeded —
and afterwards every read returned `app-error: $ must contain only plain JSON
objects and arrays`. Root cause is unknown; see Part 9. Until it is known, the
sentence must not claim what it cannot guarantee.

### Draft — the missing preconditions section

> ## Before you write anything
>
> A Canvas document is a tree of typed nodes — frames, text, icons, paths, and
> references — with themes and variables attached. The schema is not
> self-evident, it is versioned, and the version a document was created at
> constrains what you may write into it. Writing nodes you have not verified
> against the schema is the single most common way to damage a document.
>
> So, before your first write to any document:
>
> 1. `canvas documents list` — find the document and note its version.
> 2. `canvas documents export --document-id <id>` — read its actual structure.
>    Node types, property names, and the shape of nested children are all visible
>    here, and they are authoritative in a way that this document cannot be.
> 3. Match the version. A document created at 2.15 is not a document created at
>    2.17. Properties valid in one may be rejected or silently mishandled in the
>    other. Do not copy a node shape out of a `.pen` file authored elsewhere
>    without checking the version it came from.
>
> **Why this matters more here than elsewhere.** Canvas has no delete operation.
> If a write leaves a document unreadable, you cannot remove it, and neither can
> the user from the agent surface. It stays in their account. There is no undo
> and no recovery path. Treat every write to an existing document as
> irreversible.
>
> **Work incrementally.** Prefer many small `mutate` calls to one large
> `execute`. After a structural change, run
> `canvas documents export --document-id <id>` and confirm the document still
> reads. A failed export is your only early warning; without it you may not
> discover damage until the user opens the document.
>
> **On a large document**, `documents.get` can exceed a response limit or time
> out. Use `documents.export` and read the file, or narrow with `selection.set`
> and `viewport.focus`. A timeout is not evidence that the document is broken.

That is 300 words. It is longer than the three sentences originally proposed for
this job, and the length is the point: it says what the schema is, why writing
without it fails, exactly how to obtain it, what version means, what happens if
you get it wrong, and why the stakes are unusually high. Three sentences would
have been the same laziness this plan exists to fix.

### Deliverables

- [ ] Inline all five `guidelines.get` topics into `INSTRUCTIONS.md`.
- [ ] Remove the `guidelines.get` operation (decision 16). Canvas ships its
      updated `INSTRUCTIONS.md` in the same pass, so there is no window in which
      the operation is gone and the content is missing.
- [ ] Add the preconditions section.
- [ ] Remove or qualify the no-partial-commit claim pending Part 9.
- [ ] Rewrite the remaining sections to the standard.
- [ ] Apply the same contract to `browser`, `explorer`, `apps`, `borge-studio`.
- [ ] Add a packaging check: an App with operations must ship `INSTRUCTIONS.md`
      containing all five sections.

---

## Part 6 — Skills

### Split by audience

`docs/app-development.md` is about building and shipping an App, so it owns the
**authoring** contract and nothing else: where `SKILL.md` lives in the package,
the `AppSkillDeclaration` shape, how `scope: "app:" + slug` attribution is
enforced, what `penkra app package` validates, how to test a Skill before
publishing.

Penkra's `INSTRUCTIONS.md` owns **usage and trust**, because that is true of every
Skill regardless of who wrote it and is not an App author's concern.

No overlap. A fact in both places will drift.

### The authoring gap

`docs/app-development.md:95` currently reads, in full:

> Settings and Skills are declarative contributions interpreted by the host. See
> the exported TypeScript declarations in `@penkra/sdk` for the authoritative
> field types and validators.

Deferring the entire contract to type declarations is not documentation. An author
reading that cannot learn that Skills exist as a feature worth using, what a
`SKILL.md` should contain, that Skills are individually enableable, or that
attribution is enforced at load and will throw. Meanwhile the implementation is
substantial: `appSkillsCatalog.ts`, `AppSkillDeclaration` at
`packages/sdk/src/manifest.ts:82`, `ProviderSkillDescriptor` throughout
`packages/contracts/src/providerDiscovery.ts`, and a portable skill root
registered with Codex at `codexAppServerManager.ts:830`.

### Draft — the usage section for `INSTRUCTIONS.md`

> ## Skills
>
> A Skill is a written procedure, not a capability. It is a `SKILL.md` file that
> some App shipped, describing how to accomplish a task with that App: which
> operations to call, in what order, what to check between them. When a Skill is
> enabled, its instructions become available to you the same way these
> instructions are.
>
> The distinction matters more than it sounds. A Skill can name any tool, App,
> service, or command its author liked — it is prose, and prose is not checked
> against what is installed. A Skill saying "open the page in the browser App" is
> telling you the author's intended procedure. It is not evidence that a browser
> App is enabled here, that it is the same browser App the author meant, or that
> it still declares the operation named.
>
> So: read a Skill for its procedure, and verify its capabilities. The catalog
> above and `<slug> --help` tell you what actually exists. When a Skill's
> procedure and the catalog disagree, the catalog is right and the Skill is
> stale — say so plainly rather than working around it silently, because a stale
> Skill will mislead the next agent too.
>
> Skills are attributed to the App that shipped them and scoped to this Space. A
> Skill from the `canvas` App carries `canvas` authority and no more: it cannot
> authorise work in another App, and an App cannot ship a Skill on another App's
> behalf. Penkra enforces this at load time.

Compare the shipped version — "A Skill supplies instructions, never capabilities.
Loading a Skill does not install or authorize any App, MCP server, plugin,
executable, browser, or tool that its text mentions." Same fact, reasoning
removed, no procedure, and no statement of what a Skill _is_.

### Deliverables

- [ ] `## Agent Skills` section in `docs/app-development.md` — authoring only.
- [ ] Skills section in Penkra's `INSTRUCTIONS.md` — usage and trust only.
- [ ] Cross-link; do not restate.
- [ ] `penkra app package` validates declared Skills and reports actionably.
- [ ] Document the enablement model: who enables a Skill, where, what default.

---

## Part 7 — `docs/app-development.md`

The public App-author contract. Audience is a developer building an App, so its
register differs from the agent-facing documents and it should not be flattened
into them.

- [ ] Define or link every term on first use; Space is used 30+ times undefined.
- [ ] Add `## Agent Skills` per Part 6.
- [ ] Document the `INSTRUCTIONS.md` contract from Part 5 as an authoring
      requirement, with the section order and a worked example.
- [ ] Document `summary` — required per App and per operation — as agent-facing
      text that appears in the catalog, so authors know it is read by agents and
      write it accordingly. Today it reads as store-listing copy.
- [ ] Audit the browser-surface inset contract. It is stated correctly at
      `docs/app-development.md:188` and `docs/app-development-internals.md:110`,
      and `penkra-apps/browser/app.js:81` implements it correctly by deduping
      before publishing. `borge-apps/borge-studio/app.js:220` does not, and
      publishes on every resize. Either the rule is not discoverable enough at the
      point of use, or it needs runtime enforcement. Prefer enforcement.
- [ ] Check whether operations lacking a delete counterpart is a pattern; if so,
      say so in the authoring guidance rather than letting each author rediscover
      it.

---

## Part 8 — Repository hygiene

### These are two unrelated problems

Stated explicitly because they were previously conflated under a "lifecycle" theme
that sounded like an insight and predicted nothing:

**Missing `documents.delete` in Canvas** is an App API completeness gap in
`penkra-apps/canvas`. One author enumerated create/read/update and stopped. Fixed
by adding one operation. Tracked in Part 9.

**3.3 GB of scratch roots** is a development-tooling gap in this repository. QA
and sideload flows create working directories and nothing removes them. Fixed in
scripts and a contributor rule. Tracked here.

Different repositories, different owners, different fixes.

### Actions

- [ ] Delete all sixteen `.penkra-*` roots. They are gitignored and untracked; the
      ignore rule is a safety net, not a policy.
- [ ] Audit `release-local/` (311M), `.tmp/` (10M), `qa-evidence/`, `design-review/`
      for what is reproducible and what is a record worth keeping.
- [ ] Remove unrelated artifacts from the parent workspace directory:
      `admin-schoolbaseapp-com-titan-2026-07-31.tar.gz`,
      `ceo-studentsindemand-com-titan-2026-07-31.tar.gz`,
      `export-titan-mailbox.pl`, `finish-megachapel-email-migration.command`, and
      the two dated desktop staging directories.
- [ ] Make QA and sideload scripts clean up their own roots, including on failure.
      A rule that depends on a human remembering will produce this state again.

### Rule for `penkra/AGENTS.md`

This file is the contributor build document — audience is someone working on
Penkra itself, alongside `bun fmt` requirements and version authority. It is not
the agent-facing chat surface and should not be rewritten in that register. It
gains one rule, written to match its existing voice:

> ## Leave no scratch behind
>
> QA, sideload, and staging work create throwaway roots. Put them under
> `.penkra/scratch/<task-slug>/` and delete them when the task ends, including
> when it ends in failure. A scratch root that outlives its task becomes
> indistinguishable from real state: the next person cannot tell whether
> `.penkra-left-rail-qa` is a live fixture or abandoned work, so they leave it,
> and the directory is permanent. If a root must survive a task, record why in a
> `README.md` inside it.

- [ ] Add the rule.
- [ ] Add the boundary rule from Part 3: adapters carry no prompt prose.
- [ ] Separately review `AGENTS.md` on its own terms — it is dense and
      prohibition-first, which may be appropriate for its audience. Decide
      deliberately rather than by default.

---

## Part 9 — Structural bugs surfaced by this audit

Separated from the writing work because they are code defects with measurable
fixes. No writing improvement compensates for any of them.

### 9.1 Canvas document corruption — root cause unknown

A healthy document became permanently unreadable after one `Insert` of roughly 150
nodes. Verified healthy immediately before (export succeeded; `return 1+1;`
succeeded). Afterwards every read returns `app-error: $ must contain only plain
JSON objects and arrays`.

Ruled out by probes on a disposable document: nested inserts, `Object.assign`,
`padding: [0, 16]`, `cornerRadius: 9999`, `justifyContent: "space_between"`, and
dangling variable references — the last produces only a soft issue, "Variable $fog
was not found. The original $fog reference is preserved."

Remaining hypotheses: duplicate node IDs; a specific property combination; a
single-`Insert` size limit; `fontWeight` as string versus number.

Severity is high because it is silent, permanent, and unrecoverable — Canvas has
no delete operation, so a damaged document stays in the user's account. Two exist
there now: `bbae45e7-a867-42c6-9727-af47f4644c23` and
`54b69ea8-9b03-4b45-816b-772c989d1b89`.

- [ ] Bisect a reproducing `Insert` down to a minimal case.
- [ ] Determine whether writes are transactional as `INSTRUCTIONS.md` claims. If
      not, either make them so or correct the claim.
- [ ] Add validation that rejects a document-invalidating write before commit.

### 9.2 Canvas has no delete operation

Fourteen operations, none of which removes a document. Combined with 9.1, an agent
error is unrecoverable by any means available to the agent or to the user through
the agent surface.

- [ ] Add `documents.delete`, with a confirmation contract appropriate to an
      irreversible action.
- [ ] Audit every App for create/read/update operations lacking a destroy
      counterpart.

### 9.3 `documents.get` unbounded

Failed twice in one session: a timeout on one document, and 96,757 characters — a
token-limit overrun — on another. Neither failure told the caller which limit was
hit or what to do instead.

- [ ] Paginate or bound the response.
- [ ] Make the error name the limit and the alternative — "document is 96,757
      characters, exceeding the 40,000-character response limit; use
      `canvas documents export` and read the file, or narrow with
      `canvas selection.set`."

### 9.4 `documents.execute` script ceiling

100,000 bytes. Structured input from Part 4 removes the encoding pain of large
scripts but not the limit.

- [ ] Confirm the ceiling and whether a file-handle path is warranted.
- [ ] Ensure the error names the limit and the actual size.

### 9.5 Browser surface insets published on every resize

`penkra-apps/browser/app.js:81` dedupes by signature before publishing.
`borge-apps/borge-studio/app.js:220` does not, and the documented rule — report
only when structural edges change, never stream measured dimensions — is stated in
prose that an author can miss.

- [ ] Enforce in the SDK: dedupe by signature host-side, or reject streaming
      updates with an actionable error.
- [ ] Fix `borge-studio`.

### 9.6 `ClaudeAdapter.ts:4661` hardcodes capability

```ts
instructions: renderPenkraHarnessPolicy({ gatewayControlAvailable: true }),
```

against the computed value at line 970. On a degraded session Claude receives
contradictory statements about its own capabilities. Resolved by Part 2 collapsing
delivery, or by the degraded-variant decision in Part 3 — whichever lands first.

### 9.7 Error messages generally

Observed during this session, each a dead end:

| Message                                                      | Missing                                    |
| ------------------------------------------------------------ | ------------------------------------------ |
| `App command timed out.`                                     | which limit, what to try instead           |
| `$ must contain only plain JSON objects and arrays`          | which node, which property, what was found |
| `--input must be valid JSON.`                                | parse position, received value             |
| `Command expansion is not supported by penkra_exec_command.` | which character, and that no escape exists |

Anthropic's tool-authoring guidance treats actionable errors as a primary lever on
agent behaviour: an error that names the constraint and the alternative lets an
agent recover on the next call instead of guessing or abandoning the approach.

- [ ] Audit every error string reachable from an agent-invoked operation.
- [ ] Standard: name what was received, name the constraint, name the next action.

### 9.8 Three folder-move paths, one of which skips the collision check

A folder is moved into a Space by three different commands, chosen by how the user
happened to trigger the move, and they do not agree on what is legal.

**Path 1 — `space.projects.assign`** (`decider.ts:481`), the multi-select path.
Builds the destination's folder-name set and rejects a collision:

```ts
const destinationFolderNames = new Set(
  readModel.projects
    .filter(
      (project) =>
        project.deletedAt === null &&
        (project.kind ?? "project") === "project" &&
        project.spaceId === command.spaceId,
    )
    .map((project) => normalizeEntityName(project.title)),
);
// ...
if (destinationFolderNames.has(normalizedFolderName)) {
  return (
    yield *
    new OrchestrationCommandInvariantError({
      commandType: command.type,
      detail: `A folder named '${project.title}' already exists in this Space.`,
    })
  );
}
```

**Path 2 — `sidebar.item.move`** (`decider.ts:547`), the drag path. Also checks,
through a shared helper that additionally excludes the moving folder from its own
comparison (`decider.ts:598`):

```ts
yield *
  requireFolderNameAvailable({
    readModel,
    command,
    name: movedProject.title,
    spaceId: targetSpace.id,
    excludeProjectId: movedProject.id,
  });
```

**Path 3 — `project.meta.update` with a `spaceId`**, which is what the web calls
from `moveProjectToSpace` (`apps/web/src/lib/spaces.ts:121`). It validates kind,
workspace root, Space existence, archive state, and the legacy Home row — and
contains **no name check at all**. Confirmed by grep over the whole
`project.meta.update` case: two invariant calls, `requireSpaceAssignableProject`
and `requireSpace`, neither of which compares names, and no reference to
`normalizeEntityName` or `requireFolderNameAvailable`.

Note also that paths 1 and 2 do not agree with each other. Path 1 inlines the check
and has no `excludeProjectId`, so re-assigning a folder to the Space it is already
in relies on the earlier `project.spaceId === command.spaceId` continue to avoid a
self-collision. Path 2 excludes explicitly. Two implementations of one rule, and a
third caller that has neither.

**Failure:** a Space holds a folder named `Research`. Moving a second `Research`
into it through the folder's context menu or properties (path 3) succeeds, and the
Space now holds two folders that normalise to the same name. Dragging that same
folder in, or moving it as part of a two-item selection, is rejected. The rule the
other two commands exist to enforce is bypassed by choosing a different gesture.

Decision 23 resolves it by collapsing all three into one `folder.move`, rather than
copying the check into the third. A rule with three implementations has already
diverged twice; adding a fourth copy is the same bet again.

- [ ] One `folder.move` command taking one or more folder IDs; single is `n = 1`.
- [ ] `folder.update` (post-decisions 22/24) stops accepting `spaceId`.
- [ ] `sidebar.item.move` keeps ordering only; cross-Space moves delegate to
      `folder.move`.
- [ ] One collision implementation, `requireFolderNameAvailable`, with
      `excludeProjectId` always supplied.
- [ ] Regression tests: move one colliding folder through each former path.

### 9.9 `sidebar.item.move` contains an unreachable branch and a dead fallback

The same arm runs from `decider.ts:547` to `:777` — 230 lines handling folder
reorder, folder cross-Space move, thread reorder, and thread reparent in one
switch case. Two things inside it cannot execute.

At `:612` the arm rejects moving a thread to a Space:

```ts
if (movedThread && command.target.kind === "space") {
  return (
    yield *
    new OrchestrationCommandInvariantError({
      commandType: command.type,
      detail: "Threads must be moved into a folder, not directly into a Space.",
    })
  );
}
```

At `:742` it then handles `movedThread` by falling back to a chat container when
there is no target folder:

```ts
const destinationProject =
  targetProject ??
  readModel.projects.find((p) => p.deletedAt === null && p.kind === "chat") ??
  null;
if (!destinationProject) {
  return yield* new OrchestrationCommandInvariantError({
    commandType: command.type,
    detail: "No managed chat container is available for a loose Space thread.",
  });
}
// ...
spaceId: targetProject ? null : targetSpace.id,
```

`movedThread` implies `target.kind !== "space"` by the guard at `:612`, and the
only other variant is `"project"`, so `targetProject` is always non-null here.
Therefore the `??` chain never reaches the chat container, the "No managed chat
container is available" error is unreachable, and the ternary always evaluates to
`null`.

This matters beyond tidiness for two reasons. First, that unreachable error string
was cited earlier in this plan as evidence that loose threads are a live concern in
this arm; it is not evidence of anything, and Part 11.2 has been corrected. Second,
dead defensive code reads as a live invariant to the next person, who will preserve
it during the 11.2 migration and carry a chat-container reference into a codebase
that no longer has chat containers.

- [ ] Delete the fallback, the unreachable error, and the always-null ternary.
- [ ] Split the arm: reordering and reparenting are different operations with
      different invariants and should not share a 230-line case.
- [ ] Sweep the rest of the decider for branches guarded unreachable by an earlier
      check in the same arm.

---

## Part 10 — Surfaces not yet audited

Recorded so the plan is honest about its own coverage. Each was discovered while
verifying something else, and none has been examined properly.

### 10.1 `penkra context` is undocumented

```json
{
  "harness": { "name": "Penkra", "policyVersion": "2026-08-06.1" },
  "caller": {
    "threadId": "1a68ba53-...",
    "turnId": "turn:f2cc27eb-...",
    "provider": "claudeAgent",
    "projectId": "63fcd403-..."
  },
  "capabilities": {
    "threadRead": true,
    "threadCreate": true,
    "threadWait": true,
    "diagnostics": true
  }
}
```

Three problems.

The injected policy never mentions this command, so an agent does not know it can
learn its own thread ID, turn ID, project, or provider. Several policy bullets talk
about "the caller Thread" without saying how to find out which Thread that is.

`capabilities` here is a real, structured, per-capability report — and it is
exactly what `gatewayControlAvailable` is trying to express as a single boolean in
a different mechanism. Two parallel capability systems. The structured one is
better and the boolean should probably be derived from it or deleted in its favour.

`policyVersion` is returned here as well as embedded in the policy marker, so the
Part 2 decision to drop the version from prose does not fully retire it. Decide
whether it stays as machine-readable metadata or goes entirely.

- [ ] Document `penkra context` in `INSTRUCTIONS.md`, with when to call it.
- [ ] Reconcile `capabilities` here with `gatewayControlAvailable` in Part 3.
- [ ] Decide `policyVersion`'s fate in this payload.

### 10.2 `penkra capabilities` returns 59.5 KB

Measured this session: the response exceeded the inline tool-result limit and had
to be written to disk and parsed with a script. It contains `targetConstruction`
for all nine providers, a `providers` array with per-model option matrices, and
`limits`.

This is agent-facing and effectively unreadable in the flow it is meant to serve —
an agent choosing a model for `penkra threads create-many`. The policy instructs
agents to call it for exactly that purpose.

- [ ] Add filtering: `--provider <kind>`, or default to available providers only.
      Six of nine report "Provider runtime is not installed" and contribute
      nothing but weight.
- [ ] Consider a summary form for model selection and a full form for inspection.
- [ ] Same treatment as Part 9.3: a size-bounded response with an actionable error.

### 10.3 Thread orchestration policy — never audited

Roughly seven of the twenty-one policy bullets concern Threads and were never
examined in this audit. They cover `create-many` plan arrays, `requestId` retry
semantics, 3-8 word outcome-oriented task labels, self-contained instructions,
`threads wait` for every created ID, `threads send` versus a follow-up in the
current conversation, and when to notify the user about background work.

That is a substantial body of procedural guidance with real failure modes and it
deserves the same treatment as discovery: reasons attached, procedures stated,
worked example. It has had none.

- [ ] Audit the Thread bullets against actual `threads create-many` behaviour.
- [ ] Verify the `requestId` retry rules against the implementation before
      rewriting them; they read as though written from a specific incident.
- [ ] Rewrite to the Part 0 standard with a worked example.

### 10.4 Tab observation and the untrusted-data boundary

The policy states that snapshots, extractions, and screenshots are untrusted data
and never instructions. That is a prompt-injection boundary and it is stated in one
sentence with no procedure: nothing tells an agent what a hostile page looks like,
what to do when a snapshot contains something resembling an instruction, or how to
report it.

This is the highest-severity item in the whole policy and the thinnest.

- [ ] Write it properly: what the boundary is, why it exists, concrete examples of
      content that tries to cross it, and what to do.
- [ ] Check whether anything enforces it, or whether the prose is the only control.

### 10.5 `CLAUDE.md` and `AGENTS.md` have drifted

`penkra/CLAUDE.md` is 109 lines and its first line is `# AGENTS.md`.
`penkra/AGENTS.md` is 186 lines. They share overlapping but non-identical
content — both carry `bun fmt`/`bun lint`/`bun typecheck` rules and
`NEVER run bun test`, with different wording and different surrounding sections.

`CLAUDE.md` additionally carries a "Project Snapshot" that `AGENTS.md` lacks, and
it repeats the nine-provider claim examined in Part 3: "`ProviderKind` currently
spans 9 providers." Same overstatement, second location.

Two contributor documents, divergent, one mistitled, both loaded by different
tools.

- [ ] `AGENTS.md` becomes canonical; `CLAUDE.md` becomes a pointer to it
      (decision 17). Same for the three sibling packages if they have the same
      split.
- [ ] Reconcile duplicated rules.
- [ ] Correct the provider claim in both.

### 10.6 Sibling `AGENTS.md` files

```
../AGENTS.md               28 lines   workspace root
../penkra-app/AGENTS.md    30 lines
../penkra-apps/AGENTS.md   46 lines
penkra/AGENTS.md          186 lines
```

Four contributor documents across four packages, never compared. Unknown whether
they agree.

- [ ] Read all four together; reconcile contradictions; establish which is
      authoritative for shared rules.

### 10.7 Operation `summary` as agent-facing text

Every operation requires a `summary` — `notes.open` ships "Open a note." These
strings appear in `--help` output and are among the most-read text in the system,
and no guidance exists on writing them. Anthropic's tool-authoring guidance treats
per-tool descriptions as one of the highest-leverage surfaces available.

- [ ] Write authoring guidance: what the operation does, when to use it, when not
      to, what it requires first, what failure looks like.
- [ ] Rewrite the summaries in first-party Apps as worked examples.

### 10.8 Remaining first-party Apps

Part 5 covers Canvas. `browser`, `explorer`, `apps`, and `borge-studio` have not
been read. `explorer` declares a single operation, `resources.open`, and its
summary is the only thing telling an agent what it is for.

- [ ] Read and rewrite each to the Part 5 contract.

### 10.9 Not examined at all

Listed so their absence is deliberate rather than accidental:

- User-facing UI copy and error messages — this audit covered only agent-facing
  text. A user seeing "App command timed out." has the same problem an agent does.
- `README.md`, `CONTRIBUTING.md`, `docs/README.md` as a documentation map.
- Onboarding and first-run text.
- `docs/app-development-internals.md` beyond the browser-surface contract.
- The `penkra-website` and `penkra-backend` packages.
- Slash-command and composer-command surfaces.
- Notification and background-work copy.

---

## Part 11 — Legacy removal and vocabulary

### 11.1 The legacy Home chat container

`apps/server/src/orchestration/commandInvariants.ts:175` documents a row that
exists only for history:

> legacy Home chat containers kept `kind: "project"` — they are recognizable by
> the reserved home/chat workspace root plus their canonical "Home" title. Those
> containers are reachable from every Space, so they must never belong to one.
> The decider rejects renaming this legacy row so the signal cannot drift.

A row identified by _title string plus workspace path_ rather than by type. The
decider must block renaming it, because renaming would destroy the only signal
distinguishing it. That guard exists in at least two places —
`isLegacyHomeChatContainerRow` server-side and `isHomeChatContainerProject` in
`apps/web/src/lib/chatProjects.ts` — and both have to stay in agreement forever.

This is also what made `spaceId` nullable, which is what produced a wrong Space
definition during this audit. The cost of the legacy row is not just its own code;
it weakened a type that everything else reads.

- [ ] Migrate legacy Home chat rows to `kind: "chat"`.
- [ ] Delete `isLegacyHomeChatContainerRow` and `isHomeChatContainerProject`.
- [ ] Simplify `isOrdinarySpaceProject` to `project.kind === "project"`.
- [ ] Remove the decider's rename block.
- [ ] Make `spaceId` non-nullable for `kind: "project"` rows.

### 11.2 Managed chat containers — remove them

Raised alongside the legacy row, and the earlier draft of this section argued they
were current mechanism worth keeping. That was wrong on the product question. The
owner's rule is that there are no loose chats: every thread lives in a folder and
every folder lives in a Space. A container that exists to hold threads belonging to
no folder has nothing to hold.

What exists today. `ContainerKind` is `["project", "chat"]`
(`packages/contracts/src/orchestration.ts:461`). The `"chat"` variant marks a
system-owned container that is reachable from every Space and belongs to none,
which is the sole reason `spaceId` is nullable. The membership rule at
`apps/web/src/lib/spaces.ts:21` is written around it:

> Spaces organize ordinary projects only: managed chat containers are reachable
> from every Space and so belong to none. This is the membership rule the whole
> feature turns on — the sidebar list, the tab activity dots, the pickers, and the
> shortcut targets all have to agree on it.

Read that sentence as a cost rather than a description. Four independent surfaces
have to agree on one exception, and each is a place the exception can be forgotten.
`decider.ts:747` carries a matching failure — "No managed chat container is
available for a loose Space thread" — and `threadReadTools.ts:178` silently filters
the category out of the agent's folder listing. That last one is the worst of the
three: an agent receives a list that is not the whole list and is told nothing.

Removing the kind removes the exception from all four surfaces at once, and it is
what allows `spaceId` to become non-nullable, which is what allows the Space
definition in Part 1 to be stated without a caveat.

This has a migration cost that 11.1 does not, because real threads live in these
containers today and need a destination folder. That looked like an open product
decision; it is not, because the data already answers it.

`OrchestrationThread` carries its own `spaceId`
(`packages/contracts/src/orchestration.ts:675`), and `decider.ts:756` sets it with
exactly this line:

```ts
spaceId: targetProject ? null : targetSpace.id,
```

A thread's `spaceId` is populated **only when the thread is loose**. Filed threads
get `null` and inherit their Space through their folder. So the field is not "which
Space is this thread in" — it is "which Space would this thread be in, if it were
anywhere." Every loose thread therefore already records the Space it belongs to,
which is the only fact the migration needs. (The inversion is itself an argument
for this whole removal: a field that means one thing when null and another when
set is a field that will be misread.)

**Decision 31:** create one folder named `Chats` per Space that has loose threads,
and move each thread into the `Chats` folder of the Space its own `spaceId` names.
No thread changes Space, nothing needs a user prompt, and the result is visible and
renameable like any other folder — which is the point of removing the special case
in the first place. Threads whose `spaceId` is somehow null despite being loose go
to the default `"personal"` Space; log the count rather than failing the migration.

- [ ] Migrate loose threads into a per-Space `Chats` folder keyed on thread `spaceId`.
- [ ] Delete the chat containers themselves once empty.
- [ ] Drop `spaceId` from `OrchestrationThread`; a thread's Space is its folder's.
- [ ] Delete `ContainerKind` and every `kind` field, filter, and branch reading it.
- [ ] Delete the `decider.ts:747` loose-thread failure path.
- [ ] Delete the `threadReadTools.ts:178` filter so the agent's listing is complete.
- [ ] Rewrite `apps/web/src/lib/spaces.ts:21` and `isOrdinarySpaceProject`, which
      have no remaining work once both exceptions are gone.
- [ ] Make `spaceId` non-nullable.

### 11.3 Folder, not project — settled

The UI calls them folders. The contracts call them projects or containers. Both
words are load-bearing today:

```
apps/web/src/components/Sidebar.logic.ts:54   beginInlineFolderCreation
apps/web/src/components/Sidebar.logic.ts:57   openInlineFolderCreator
apps/web/src/components/Sidebar.logic.ts:174  canArchiveSidebarFolder
packages/contracts/src/orchestration.ts:464   OrchestrationProject
packages/contracts/src/orchestration.ts:977   "Required for ordinary folders and
                                               absent for managed chat containers"
```

That last line uses both words in a single comment, which is the clearest evidence
that no one word is winning on its own.

**Folder wins** (decision 22). It is the word the user sees, the word the user
says, and therefore the word an agent hears in the request it has to act on. Every
translation step between what a user says and what an agent calls is a place the
agent can pick the wrong thing, and the decider's own error strings have already
drifted to the user's word — `"A folder named '…' already exists in this Space."`
is raised by a command called `space.projects.assign`. The contracts are the layer
that is out of step.

The rename is agent-facing, not cosmetic. `penkra projects list` becomes
`penkra folders list`, which is a breaking change to the command surface and has to
land with Part 2's document rather than after it.

- [ ] `OrchestrationProject` → `OrchestrationFolder`; `ContainerId` → `FolderId`.
- [ ] `project.create` / `project.meta.update` / `project.delete` →
      `folder.create` / `folder.update` / `folder.delete` (decision 24 drops `.meta`).
- [ ] `penkra projects list` → `penkra folders list`.
- [ ] Event names, read-model fields, and test fixtures follow.
- [ ] Sweep for the losing word: `grep -ri "project" packages/contracts apps/server/src/orchestration`
      should return only genuine unrelated uses afterwards.
- [ ] `space.projects.assign` is **parked** (decisions 21 and 23); when it is
      unparked it becomes `folder.move`, not `space.folders.assign`. See 11.4.

### 11.4 Command naming: drop `.meta`, drop `reorder`, name the subject

Three corrections to the earlier draft of this section, in increasing order of how
wrong I was.

**Scope.** These are not agent-facing commands. `space.*` and `project.*` are
dispatched on the internal shell command bus by the web UI; grepping
`appRuntimeCli.ts` for `space.` returns nothing, and the agent's core command list
is threads, apps, tabs, open, and the App-development commands. Nothing in this
section changes what an agent types. It is here because Part 1's definitions must
describe the model these commands implement, and because 9.8 and 11.3 rewrite them.

**`reorder` goes** (decision 21, now parked). The earlier argument for keeping it —
that position is a concern distinct from display metadata — proves too much: by
that reasoning every mutable field deserves its own command. Position is not a
lifecycle state, it has no invariant beyond staying within the sibling set, and
`space.reorder` is a thin translation of a drag gesture into a
`{ before | after, spaceId }` pair (`apps/web/src/lib/spaces.ts:100`). It folds into
the update command. One consequence to carry, not discover: drag-to-reorder is a
live feature with a call site and two tests, so `sortOrder` moves into the update
payload rather than being dropped.

**`.meta` goes too** (decision 24). I defended it twice and neither defence holds.

The first defence was "four unrelated failure modes in one payload," which was an
assertion with no example. The second was "a reader can predict a command's blast
radius from its name." That one is worse, because it sounds like a reason and
isn't. `space.update` predicts blast radius exactly as well as `space.meta.update`
does — in both cases the reader's actual question is _which fields does this
accept_, and in both cases the only honest answer is the schema. The segment adds
a word and answers nothing.

Worse, `meta` is not a thing in this system. There is no meta entity, no meta
field, no meta concept anywhere in the contracts. It is a category label for
"fields I have decided are boring," and category labels drift: the moment someone
adds a field that is arguably boring, it lands in the `.meta` command, which is
precisely how `spaceId` and `kind` ended up inside `project.meta.update`. The
segment did not prevent that. It provided cover for it.

And the reading the owner gave is the one a newcomer will give: `project.update`
obviously means updating the existing project. That is what the command does. The
extra segment invites the reader to wonder what _other_ kind of update exists —
and the answer is none.

**What the real fix was.** The eight-branch decider arm on `project.meta.update`
is a payload problem, not a naming problem:

1. is the `kind` changing
2. was a `workspaceRoot` set on a virtual container
3. does the target Space exist
4. is the resulting `spaceId` null for an ordinary folder
5. is it non-null for a chat container
6. was an archive flag set on a non-folder
7. is a deleted folder being archived
8. is this the legacy Home row

Nearly every field is optional and a typical call sets one, so a caller cannot tell
which of the eight they might trip, and the failure does not say which axis failed.
No prefix fixes that. Removing `spaceId` (decision 26), `kind` (11.2), and the
legacy row (11.1) deletes branches 1, 3, 4, 5, and 8 outright, and what remains is a
command that genuinely only updates a folder — at which point it should simply be
called `folder.update`.

**When a segment is earned — second attempt.** The first attempt said a segment is
earned when it "names a distinct addressable sub-resource," and offered
`space.folders.assign` as the earned case because the payload carries `folderIds`.
That does not survive contact with the data model. There is no `folders` collection
stored on a Space; membership lives on the folder row as `project.spaceId`. The
segment names a _view_, not a resource — you cannot address it, there is no
`space.folders.get`, and the identities in the payload are top-level folder IDs
that exist perfectly well without any Space. So `folders` fails for the same reason
`meta` fails: it names something that is not there.

**Third attempt, and the reason the first two failed.** Both earlier attempts drew
their examples from `thread.marker.*`, `thread.message.*`, `thread.turn.*`, and
`thread.session.*`. Those are internal bus commands. **None of them is an
agent-facing operation**, so none of them is evidence about the surface this plan
exists to fix. The rule may well have been correct; the demonstration proved
nothing about the thing being fixed. `thread.marker.*` is also being pruned
outright (decision 25), which would have left the rule's flagship example pointing
at a deleted feature.

Re-derived on the surface that is actually in scope. The agent-facing shape is
`<subject> <verb>`, not `<parent>.<child>.<verb>`:

| Agent-facing command                                 | Shape                          | Nested? |
| ---------------------------------------------------- | ------------------------------ | ------- |
| `penkra threads list`                                | subject + verb                 | no      |
| `penkra tabs snapshot --tab-id <id>`                 | subject + verb, target by flag | no      |
| `canvas documents create`                            | subject + verb                 | no      |
| `penkra app access invite --app-id <id> --email <e>` | parent + child + verb          | **yes** |

There is exactly one genuinely nested agent-facing command family:
`penkra app access invite` / `list` / `revoke` (`appRuntimeCli.ts:313`). It is
earned — an invitation exists only against a specific App, carries no meaning
without it, and is not reachable by ID alone. That is the whole earned set.

Everything else on the agent surface is `<subject> <verb>` with targets supplied as
flags, which is the right default: it keeps the command name short, keeps the
target explicit, and gives an agent one shape to learn instead of two.

**One retraction to record in place.** An earlier draft argued that folders differ
from markers because deleting a Space leaves orphaned folder rows. It does not.
`decider.ts:452` refuses:

> "Move every folder and chat thread out of this Space before deleting it."

`project.delete` does the same one level down via `requireProjectHasNoThreads`
(`decider.ts:1057`). Both levels refuse to delete a non-empty container rather than
cascading, so the hierarchy is strictly maintained and orphans are unreachable. The
model is already correct; that sentence invented a hazard the system does not have.
The distinction that actually holds is **addressability**, a static fact about the
schema: `requireProject` finds a folder by ID alone, with no Space in hand, and
nothing can find a marker that way.

The practical consequence for App authors is a single sentence rather than a
taxonomy: **name the operation for the thing it changes, pass targets as inputs,
and only nest when the child cannot be addressed without its parent.**

- [ ] `space.meta.update` → `space.update`; `project.meta.update` → `folder.update`;
      `thread.meta.update` → `thread.update`. Leave every other `thread.*` segment
      in place.
- [ ] `SpaceReorderCommand` removal and `space.projects.assign` → `folder.move` are
      **parked** (decisions 21 and 23). Recorded here so the naming argument is not
      lost when they are unparked.
- [ ] After 11.2 and decision 26, assert `folder.update` accepts display fields only
      and that branches 1, 3, 4, 5, and 8 above are gone.
- [ ] Sweep for the segment: `grep -rn "\.meta\." packages/contracts apps/server/src/orchestration apps/web/src`
      should return nothing.
- [ ] Document the earned-segment rule in `docs/app-development.md` as **guidance,
      not validation** (decision 29). Give worked examples an author can
      pattern-match against — `documents.create` and `documents.publish` as the
      ordinary shape, `documents.comments.add` as a nesting that may be earned if a
      comment ID is meaningless without its document, `documents.meta.update` as
      nesting on an adjective — and state the reasoning so an author can judge a
      case none of the examples cover. The evidence base is one command family
      (`penkra app access *`), which is too thin to reject anyone's design over.

---

## Part 12 — The Thread command surface

Fourteen agent-facing commands, and until now this plan did not mention them. That
is the largest coverage gap in it. The Apps surface got Parts 4 and 5; the Thread
surface — which every provider sees on every turn, whether or not any App is
installed — got a single policy bullet.

The commands, from `AgentGateway.test.ts:169`:

```
penkra threads list                penkra threads create
penkra threads read                penkra threads create-many
penkra threads activity            penkra threads send
penkra threads events              penkra threads interrupt
penkra threads runtime-events      penkra threads rename
penkra threads diagnose            penkra threads archive / unarchive
penkra threads retry-projection    penkra threads wait
```

### 12.1 `create-many` is deleted, and so is its compensating saga

Decision 27 removes `penkra threads create-many`; an agent that needs five threads
calls `create` five times. The contract divergence that motivated this is real —
the singular tool made `target` optional and accepted flat
`provider`/`model`/`options` alongside it, which the plural rejected — and one
command removes it entirely.

What has to be said plainly is what else goes. `create-many` is not a loop wrapper.
`creationCoordinator.ts` is 791 lines implementing a compensating saga:

- a deterministic `operationId` derived from the caller and the plan
  (`:280`, `stableGatewayDigest`);
- deterministic per-index IDs including a `compensateCommandId` reserved up front
  for each thread before any is created (`:414`, `makeAgentCreationIds`);
- `compensateClaimedOperation` (`:423`), which on failure marks the operation
  compensating, issues each reserved compensate command, counts successes, and
  collects per-thread errors;
- a persisted operation store with `markCompensating` / `markTaskStatus`, so a
  failed compensation is recorded as `failed` rather than left looking active;
- `startupRecovery.ts`, so an operation interrupted by a restart is resumed rather
  than abandoned;
- `redactCreationPlanForPurgedCaller` (`operationPlan.ts:41`), so a stored plan
  survives the caller being purged.

That is the machinery behind the tool description's claim: "Validation or preflight
failures create nothing and may be corrected with the same requestId; durable
retries replay the exact operation."

**Five separate `create` calls cannot offer that.** Call 3 failing leaves threads 1
and 2 in the sidebar, and nothing rolls them back, because rollback needs a plan
recorded before the first create. The user asked for five threads and has two, with
no operation to retry and no record tying them together.

This is a real trade and it is the owner's to make; it is recorded here so it is
made knowingly rather than discovered later.

**A7 — decided: accept partial creation.** The saga goes with the tool.

The trade is deliberate. Rollback is real machinery, but it exists to serve a batch
command that no longer exists, and preserving it would mean keeping 791 lines of
compensation, an operation store, and a startup-recovery path alive for a caller
shape nothing produces. The cost is that a failure part-way through leaves earlier
threads standing.

That cost is only acceptable if the failure says so. An agent told "thread creation
failed" after three of five succeeded will most likely start over and create three
duplicates. An agent told which threads exist can finish the job. So the error text
is not a nicety here — it is the thing standing in for the rollback, and it belongs
to the same writing standard as everything else in this plan (Part 0, and 9.7 on
actionable errors).

- [ ] Delete `create-many`, `creationCoordinator.ts`'s compensation half,
      `operationPlan.ts`, and `startupRecovery.ts`'s creation path. Remove the
      operation-store rows. Leave nothing unreferenced (Part 8).
- [ ] On any `create` failure, name the threads that already exist and their IDs.
- [ ] Rewrite the `create` description: no reference to a deleted tool, and an
      explicit statement that creation is not atomic across calls.
- [ ] Fold the singular schema down: `target` required, no flat
      `provider`/`model`/`options`.
- [ ] Rewrite the description so it no longer points at a deleted tool.

### 12.2 The tool annotations are wrong, and inconsistently wrong

`toolRuntime.ts:20` defines the shared constant:

```ts
export const WRITE_TOOL_ANNOTATIONS = {
  readOnlyHint: false,
  destructiveHint: true,
  idempotentHint: false,
  openWorldHint: false,
} as const;
```

Every write tool uses it — except the two create tools, which spell their
annotations out inline and deviate on two of the four fields (`idempotentHint:
true`, `openWorldHint: true`). Nothing documents why creation would be idempotent
and open-world when renaming is neither.

Beyond the inconsistency, the shared constant itself is wrong in one place that
matters. `destructiveHint` is meant to indicate that a tool may perform destructive
or irreversible updates; it is applied here to `penkra_create_thread`,
`penkra_create_threads`, and `penkra_set_thread_title`. Creating a thread destroys
nothing and renaming one is trivially reversible. Meanwhile `penkra_interrupt_thread`
— which terminates an in-flight turn and is the most genuinely disruptive tool in
the set — carries the identical annotation, so a host that uses these hints to
decide what to auto-approve cannot distinguish "make a new thread" from "kill a
running one."

The hints are host-facing metadata, not prose, but they are part of the same
writing problem the rest of this plan addresses: they are the machine-readable
summary of what a tool does, and right now every write tool claims to do the same
thing.

- [ ] Set `destructiveHint: true` only for `interrupt`, `archive`, and anything
      that ends or hides work; `false` for create, rename, send.
- [ ] Set `idempotentHint` from actual behaviour: the create tools are idempotent
      _on `requestId`_, which is a real and unusual property worth stating in the
      description as well as the hint.
- [ ] `openWorldHint: false` for all of them; a Penkra thread is local.
- [ ] Use `WRITE_TOOL_ANNOTATIONS` (or a small set of named variants) everywhere;
      no inline annotation blocks.
- [ ] Check the read and diagnostic tools for the same drift —
      `threadReadTools.ts` and `threadDiagnosticTools.ts` carry their own.

### 12.3 `archive` takes a boolean mode flag; the bus does not

`penkra_set_thread_archived` (`AgentGateway.ts:487`) is one tool with
`archived: boolean`, dispatching to two different orchestration commands:

```ts
type: archived ? "thread.archive" : "thread.unarchive",
```

The internal bus models these as two commands. The agent surface models them as one
command with a mode flag. The agent CLI then re-splits them, because
`AgentGateway.test.ts:187` renders the tool as either `penkra threads archive` or
`penkra threads unarchive` depending on the flag's value — so the same operation has
two names at one layer, one name at the next, and two again at the last.

A boolean that selects between two behaviours is a second command wearing a
costume. It also reads badly at the call site: `archive --input '{"archived":false}'`
is a command whose name and payload contradict each other.

- [ ] Split into `penkra threads archive` and `penkra threads unarchive`, matching
      both the bus below it and the CLI rendering above it.
- [ ] Apply the same test to every other boolean-mode argument on this surface.

### 12.4 `threadId` defaults to the caller, on destructive tools

`penkra_set_thread_archived` requires only `archived`; `threadId` is optional and
falls back to `context.callerThreadId`. The description says so — "Defaults to your
own thread when threadId is omitted" — which is the minimum, but the default is
still that an agent which omits an argument archives _itself_, ending its own
visibility in the sidebar mid-turn.

Defaults are appropriate for read tools, where the worst case is reading the wrong
thing and noticing. For a tool that hides a thread from the user, the argument
should be explicit, and "my own thread" should be a value the agent has to write
rather than a value it gets by omission.

- [ ] Audit every tool on this surface for caller-defaulted `threadId`.
- [ ] Require it explicitly on any tool with a non-trivial undo.
- [ ] Where a self-target is legitimate, consider a sentinel the agent must type.

### 12.5 The provider enum exposes six providers that do not exist

`penkra_create_thread` declares `provider: { type: "string", enum: [...PROVIDER_KINDS] }`
— all nine, including the six that report "Provider runtime is not installed" when
probed. Decision 14 already narrows `ProviderKind` to `codex | claudeAgent |
opencode`; this is the agent-facing consequence and should land with it, since an
enum is a promise about what the caller may pass.

- [ ] Narrow the enum with decision 14.
- [ ] Ensure the failure for an unavailable provider names the available ones.

### 12.7 `send` fabricates a user message, and nothing stops it targeting itself

This is the answer to the policy line nobody could explain:

> Use `penkra threads send` for a later manual follow-up such as "continue" on an
> existing Thread. Never use it for a manual follow-up turn that belongs in the
> current conversation.

The handler (`AgentGateway.ts:380`) builds this:

```ts
message: {
  messageId: MessageId.makeUnsafe(`agent:${suffix}:message`),
  role: "user",
  text: message,
  attachments: [],
},
dispatchMode,
dispatchOrigin: "agent",
```

Two facts follow. First, the message is written with **`role: "user"`**. It lands in
the target thread's transcript in the user's voice. Second, it goes through
`dispatchTurnStart` — it does not append a note, it **starts a turn**.

So an agent that uses `send` on its own thread does two things at once: it puts
words in the user's mouth in a transcript the user will later read, and it starts a
second turn on top of the one still running. That is the whole reason for the
prohibition, and the current policy states it without either half.

`dispatchOrigin: "user" | "automation" | "agent"` (`orchestration.ts:235`) is set to
`"agent"`, so the provenance is recorded and a UI _can_ distinguish it. Whether any
UI does is untested here.

**And nothing enforces the rule.** The handler reads `threadId` as required and
never compares it to `context.callerThreadId`. Contrast `penkra_set_thread_archived`
at `:503`, which _defaults_ to the caller. So the surface defaults to self where
self is dangerous (12.4) and permits self where self is forbidden.

- [ ] Reject `threadId === callerThreadId` in `send` with an error naming the two
      failure modes, so the rule stops depending on the agent having read a policy.
- [ ] Rewrite the policy line to state both halves: fabricated user turn, stacked
      turn. A rule with its reason attached survives paraphrase; this one has not.
- [ ] Confirm the UI renders `dispatchOrigin: "agent"` distinguishably. If it does
      not, a user cannot tell an agent-authored "user" message from their own.
- [ ] Reconsider `role: "user"` for agent-originated sends. The role is a claim
      about authorship and it is false here.

### 12.6 Not yet audited on this surface

Recorded so this Part is honest about its own coverage, in the same way Part 10 is.

- The eight read and diagnostic tools' descriptions and pagination contracts
  (`threadReadTools.ts`, `threadDiagnosticTools.ts`).
- `penkra threads wait` — the policy tells agents to call it for every created
  thread; its timeout, partial-result, and failure semantics are undocumented here.
- `penkra threads retry-projection` — appears in no policy text at all; unclear
  when an agent is expected to reach for it.
- `penkra threads send` — the policy carries a subtle prohibition ("Never use it
  for a manual follow-up turn that belongs in the current conversation") whose
  reasoning is not stated anywhere. Part 10.3 covers the bullet; this covers the
  tool.
- The relationship between `activity`, `events`, and `runtime-events` — three
  read tools whose names do not distinguish them.

---

## Sequencing

Ordered by dependency, not by size.

**Stage 1 — Unblock.** No open questions remain against the writing work. One
product decision still gates a migration: which folder existing chat-container
threads move into (Part 11.2). That blocks 11.2 only, not the sequence.

**Stage 1b — Audit the rest.** Part 10 lists surfaces this plan has not examined.
Sections 10.3 (Thread orchestration) and 10.4 (the untrusted-data boundary) are
substantial bodies of live policy that need auditing before Part 2's document can
be written, since both belong in it.

**Stage 2 — Definitions.** `docs/concepts.md`. Nothing downstream can be written
well until the nouns exist.

**Stage 3 — Command surface.** Part 4 before the documents, so `INSTRUCTIONS.md`
is written against the structured shape from the start rather than written twice.

**Stage 4 — Penkra as App zero.** Part 2. The builder, the document, the catalog,
delivery collapsed to one path per provider. This is the largest single piece and
the one that removes the discovery failure.

**Stage 5 — Provider layer.** Part 3. Drop the preset, delete adapter prose,
behavioural comparison. Separable from stage 4 but sharing the same system-prompt
authorship, so likely the same working session.

**Stage 6 — Apps.** Canvas first, since the evidence is best there. Then browser,
explorer, apps, borge-studio. Then the packaging check that makes the contract
enforceable.

**Stage 7 — Author docs.** Parts 6 and 7.

**Stage 8 — Hygiene, legacy, and bugs.** Part 8 is independent and can happen at
any point. Parts 11.1 and 11.2 should both land before Part 1's `docs/concepts.md`
is finalised: together they remove both `spaceId: null` cases, so the Space
definition can be written without exceptions rather than around them. Part 11.3
(the folder rename) must land with Part 2, because it changes a command an agent
types. Parts 9.8, 9.9 and 11.4 should land with 11.3: all four rewrite the same
decider arms, and doing them separately means touching `sidebar.item.move` three
times.

**Out of band — Part 9.1 and 9.2.** These are the only items in this plan that
destroy user data, and they are already doing so: two documents in the owner's
account are permanently unreadable with no delete path. They do not depend on any
writing work and should be scheduled immediately rather than waiting for the
sequence above.

---

## Verification

Writing changes resist automated verification, so most of this is behavioural and
must be run rather than reasoned about.

**Automated**

- [ ] Injected bytes equal `penkra --help` bytes.
- [ ] Rendered catalog equals installed Apps for a fixture Space.
- [ ] No App, operation, or command named in prose that does not exist.
- [ ] No adapter contains prompt prose.
- [ ] Every App with operations ships a five-section `INSTRUCTIONS.md`.
- [ ] Round-trip tests for `$`, backticks, quotes, newlines, nested JSON.
- [ ] Every term in `docs/concepts.md` is linked on first use elsewhere.

**Behavioural — the real test.** Run the same tasks against old and new documents
on all three live providers (codex, claudeAgent, opencode):

- [ ] "Create a new Canvas design for Borge Studio." The original failure. Success
      is discovering the `canvas` App on the first tool call.
- [ ] A task solvable by an installed App the user never names, to test whether
      discovery fires without a keyword.
- [ ] A task with two plausible Apps, to test the tie-break order.
- [ ] A task referring to something on screen, to test tab resolution.
- [ ] A Canvas write task, to test whether the preconditions section is obeyed.
- [ ] A task following a Skill that names a capability which is not installed.

**Manual.** Per `AGENTS.md`, start a fresh Penkra Dev instance and exercise the
affected flows in the desktop app. Record what was exercised and the result.

**Not verifiable, stated honestly.** Whether the writing is _good_ — whether a
competent newcomer reads it and acts correctly — is a judgement call. The
behavioural tests above are the closest proxy and should be treated as the
acceptance criterion, not the automated checks.
