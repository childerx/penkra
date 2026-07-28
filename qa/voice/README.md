# Long voice-note QA

This on-demand check exercises the real Electron microphone capture, rolling WAV
chunking, ChatGPT uploads, ordered retries, overlap merging, and composer insertion.
It is intentionally excluded from ordinary CI because the retained recording runs
for 9 minutes 15 seconds and calls the live transcription service.

The fixture is Chapter 1 of the LibriVox dramatic reading of _Headlong Hall_. It
contains a narrator and multiple character voices. Its provenance, public-domain
notice, duration, and checksum are recorded in `fixtures/SOURCE.json`.

## Prepare

From the repository root:

```sh
bun run qa:voice:prepare
```

This verifies the retained MP3 checksum and creates a 24 kHz mono PCM WAV under
`.qa-artifacts/voice/`. The generated WAV is ignored by Git.

## Run through Penkra Dev

Quit any running source-development desktop instance, then launch:

```sh
PENKRA_VOICE_QA_WAV="$PWD/.qa-artifacts/voice/headlong-hall-chapter-1-24khz-mono.wav" \
  bun run electron:dev
```

Open a project and thread, start one voice note, and let the fixture reach its end.
Chromium presents the WAV as the microphone and does not loop it. Stop the recording
after 9:15, then wait for transcription.

The explicit fixture environment also disables Chromium's process sandbox for this
QA launch because Chromium's sandboxed audio service cannot read an arbitrary local
WAV path. Never use this environment variable for normal browsing or daily work.

Pass criteria:

- Recording remains active beyond the former 2:00 boundary.
- Stop produces multiple sequential transcription requests without a duration error.
- The final composer text is coherent from the beginning through the end.
- Boundary text is not conspicuously missing or repeated.
- A transient chunk failure is retried without replaying earlier successful chunks.
- Cancel, thread navigation, or provider changes prevent stale text insertion.

This is exploratory QA, so punctuation and exact wording are not asserted. The
deterministic unit tests cover sample boundaries, overlap, ordering, isolated retry,
cancellation, WAV format, and transcript deduplication on every normal test run.
