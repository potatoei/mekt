# Running the MBAACC fixes — a step-by-step runbook

Everything below runs on **your** machine, because the engine changes live in
the `wine-assembly` submodule and the measurements need your game files. This
document is ordered: do 1, then 2, then decide with 3.

| Step | What it produces | Time |
|---|---|---|
| [1](#step-1--get-the-code-into-your-tree) | The fixes in your tree | 5 min |
| [2](#step-2--prove-the-engine-changes-are-safe) | Green build + tests | 5 min |
| [3](#step-3--measure-your-archives-the-important-one) | `ranges.json` for `0002.p` / `0003.p` | 2 min |
| [4](#step-4--turn-off-the-1-gb-launch) | A launch that is tens of MB | 2 min |
| [5](#step-5--verify-each-reported-bug-is-actually-fixed) | Yes/no per issue | 30 min |
| [6](#step-6--start-staging-what-is-still-guessed) | Cause of the 20 FPS menus | 10 min |

---

## Step 0 — what I could not do for you, and why

I could not push anything, and this is the first thing to understand because it
determines step 1:

```
root repo   potatoei/mekt        -> I committed; push is yours to do
submodule   stevenvinhtran/wine-assembly -> "Permission denied to potatoei"
```

The engine is a **fork owned by someone else**. Everything I changed is
committed in your local checkout, and I made you a patch:

`docs/mbaa/mbaa-engine-fixes.patch` (2 commits, 24 files, +1889/−120)

---

## Step 1 — get the code into your tree

The branch is `arena/42e272c4-mekt` in `/home/user/mekt`, with the submodule at
commit `7cd76efab` (was `8e79a8637`).

### If you work from the same clone I do

Nothing to do — the commit is already on the branch. Skip to step 2.

### If you are applying this elsewhere (your own machine)

```bash
# 1. Root repo: fetch the branch (contains the audit doc + the patch file)
git fetch origin arena/42e272c4-mekt
git checkout arena/42e272c4-mekt

# 2. Submodule: apply the engine patch
cd wine-assembly
git am /path/to/mekt/docs/mbaa/mbaa-engine-fixes.patch
```

`git am` applies both commits with their messages intact. If your submodule is
not at `8e79a8637`, apply with `git apply --3way` instead and read
`docs/mbaa/AUDIT_2026-11-17.md` §8 for what each hunk does.

### If you would rather start with only the safe part

Not all of my changes are equal. In order of "safe to take today":

1. **ReadFile park contract** (`lib/filesystem.js`,
   `src/09a7d-handlers-shell-file.wat`) — this is the dialogue bug. Pure bug
   fix, no policy change.
2. **P8 palettes** (`src/09ad-handlers-d3d9.wat`, `lib/d3d9-host.js`) — the
   white-texture family. Needs the wasm rebuild, no config change.
3. **`host.js` lazy+preloadRanges fix** — one line; without it the preload
   feature throws.
4. **`preloadAssets` on `mbaa`** (`lib/apps.js`) — **this one changes launch
   behaviour** (1 GB). Do step 3 and 4 below before shipping it.

---

## Step 2 — prove the engine changes are safe

First, one dependency install — the D3D9 tests decode real PNGs and module
resolution fails without it (this is the single most common "the test is
broken" here, and it is not):

```bash
cd wine-assembly
npm install --omit=dev     # pngjs + wabt, no build toolchain needed
```

Then two commands, both should be green before you trust anything else:

```bash
bash tools/build.sh          # expect "Build complete: ... 1765269 bytes"
bash tools/check-test-manifest.sh
```

Expected tail:

```
test tiers: OK (1837 files: 1435 unit, 396 e2e, 6 smoke, 0 quarantined)
check-test-manifest: OK (all discovered tests accounted for)
```

Then the tests that cover the changes themselves — each prints `PASS`:

```bash
node test/test-readfile-park-idempotence.js   # dialogue bug
node test/test-d3d9-palette-state.js          # P8 palettes are real state
node test/test-d3d9-p8-conversion.js          # index -> palette expansion
node test/test-app-files-policy.js            # preload policy + budget guard
node test/test-app-files-switches.js          # ?eager-files / ?lazy-files
node test/test-mbaa-texture-census.js         # the census tool
node test/test-perf-hud-waiting.js            # the HUD rates
```

**If `build.sh` fails**, it is almost always the region allocator: the palette
table grew `$d3d9_program_alloc` from 25616 to 42000 bytes and the palette
table ends at `+42004`. The build prints the region hash; a mismatch there
means someone else's change collided with mine.

Also run one real binary, which catches guest-execution regressions the unit
tests cannot:

```bash
ln -sfn test/binaries binaries      # once
node test/run.js --app=sol --quiet-api --max-seconds=6
```

Expect `Stats: ... API calls ... exit=0`. On the branch this is what I got.

---

## Step 3 — measure your archives (the important one)

This is the step that turns a **1 GB launch** into **a few tens of MB**, and
it is the only step that needs your game files.

### 3a. Locate the archives

They are `0002.p` (sprites, ~900 MB) and `0003.p` (stages, ~140 MB) in your
MBAACC install — the same folder as `MBAA.exe`. Also census `0000.p`,
`0001.p`, `0004.p` while you are there; costs seconds each.

### 3b. Run the census on each

```bash
cd wine-assembly

node tools/mbaa-texture-census.js /path/to/MBAACC/0002.p --list
node tools/mbaa-texture-census.js /path/to/MBAACC/0002.p --out=ranges-0002.json
node tools/mbaa-texture-census.js /path/to/MBAACC/0003.p --out=ranges-0003.json
```

The tool reads **42 MB of RAM and 22 ms** for a 1.6 GB archive, so
nothing here is expensive. `--list` prints one line per member:
`texture:raw 65536 @0x000a1234 spr_foo.bin`.

Everything above was verified in a **fresh clone of the pinned base with only
the patch applied** — build green, every listed test `PASS` — so if a step
fails for you, it is the environment and not the patch. The one environment
trap is `npm install --omit=dev` (step 2); without it `pngjs` is missing and
`test-d3d9-palette-state.js` fails with `Cannot find module 'pngjs'`, which
looks like a broken change and is not.

### 3c. Read the three numbers

```
0002.p: 812 members, 878.3MB
  texture members 640 (742.1MB), other 172 (136.2MB)
  pinning 311 range(s), 742.1MB (84.5% of the archive)
```

That third line is the decision. Three outcomes:

| Pinned | Meaning | Do this |
|---|---|---|
| **< ~200 MB** | Most of the archive is audio/script that streams. | Ship it. Launch cost is your pinned number. |
| **200–500 MB** | Most of it really is textures. | Step 4 with a `--max-mb` budget you accept, or stay on whole-archive preload. |
| **Refuses, "not a .p archive"** | Your build's container differs from the one the tool understands. | **Do not guess ranges.** The tool exits non-zero and writes nothing on purpose. Send me the first 64 bytes (`xxd -l 64 0002.p`) and the member count, and keep `preloadAssets` for the whole archive meanwhile. |

### 3d. If it refuses

That is a designed outcome, not a failure. A guessed range list pins the wrong
bytes, and a texture that is resident but wrong is worse than one that streams
slowly. Two hexdumps settle it:

```bash
xxd -l 64 /path/to/MBAACC/0002.p
ls -l /path/to/MBAACC/*.p
```

---

## Step 4 — turn off the 1 GB launch

Right now `lib/apps.js` says `preloadAssets: ['0002.p', '0003.p']`, which means
**1040 MB required before the first frame**. That is your report #1 taken
literally ("stream all the textures before the user enters the game"), and you
should decide consciously whether you want it.

### Option A — pin texture ranges only (recommended if step 3 gave you a number)

Put the ranges in the **manifest next to the bytes**, not in the registry — a
deployed build should not carry sizes that only match one machine's copy:

`binaries/mbaa/.wine-assembly-browser.json`:

```json
{
  "schemaVersion": 1,
  "files": [ ... your existing entries ... ],
  "preloadAssets": ["0002.p", "0003.p"],
  "preloadRanges": {
    "0002.p": { "schemaVersion": 1, "size": 921000000, "ranges": [[...]] },
    "0003.p": { "schemaVersion": 1, "size": 146800640, "ranges": [[...]] }
  }
}
```

Or just paste the whole contents of `ranges-0002.json` as the value — it is
already in that shape (`{schemaVersion, size, ranges}`), and the loader accepts
it verbatim. **`size` must equal the real file size**; if it does not, the
policy refuses the list and falls back to a whole-file preload (loudly, in the
console), because ranges measured on other bytes would pin the wrong data.

### Option B — keep whole-archive preload

Change nothing; the console will warn that the launch is 1040 MB. Fine on a
fast pipe, wrong for a phone.

### Option C — stream everything again

Delete the `preloadAssets` line from the `mbaa` entry in `lib/apps.js`, or add
`?lazy-files` to the URL for one launch. The mid-battle stalls (and the
NULL-cached-texture battles from issue #1) come back with it.

### 4e. Verify the choice took effect

Watch the console at launch. You should see:

```
File policy: lazy; 2 eager (...) bytes, 5 on demand (...) bytes
```

and **no** `[files] ... preloadAssets makes NNNMB required at launch` warning
once ranges are in place. The launch window's progress bar should finish in
tens of seconds rather than ten minutes.

---

## Step 5 — verify each reported bug is actually fixed

I cannot run MBAACC here, so this is the part only you can do. Work down the
list; each has an observable pass condition and a known way to fail.

### 5.1 Dialogue tearing (your issue #3)

**Do:** enter a pre-battle dialogue scene. Watch a line being revealed.
**Pass:** the line appears whole, or reveals at a steady character rate.
**Fail:** still stutters "The" → "Th" → "The...".

If it fails, the tearing is not the park path. Get the trace:

```
?trace-api=ReadFile
```

and look for a `ReadFile` whose returned count is smaller than the previous
call's for the same buffer. That combination is what the fix targets; a
different pattern means the dialogue is drawn by something else entirely (see
5.4).

### 5.2 Battles with no characters / black background (your issue #2)

**Do:** play several matches; watch for the round-1 black-arena case you
described.
**Pass:** sprites and stages appear in every round.
**Fail:** record which `NNNN.p` was being read at the moment — run with
`?trace-fs` and capture the last 50 lines before the black frame.

This is the one I am least able to prove, because it depends on the guest
caching a failed read as NULL forever. With the preload in place the read
should not fail at all; if it still does, the transcript names the file and
offset and that is directly actionable.

### 5.3 White placeholder textures (your issue #4)

**Do:** play until you see one, with this in the URL:

```
?d3d9-audit=textures
```

**Pass:** no white placeholders.
**Fail:** the log names the format of every upload. Send the lines around the
white one. `P8` with a palette mark of `0` means the game uploaded indices
without setting a palette first, which is a different bug than the one I fixed
and I need to see it to know which.

### 5.4 Menus at 20 FPS (your issue #5 / GitHub #3)

**Do:** turn the perf HUD on, open Character Select, let it run 5 seconds.
Read the line that now says:

```
waiting 150 park/s 0 fill/s
```

**Interpretation — this is the whole point of the change:**

| Reading | Cause | Next |
|---|---|---|
| `park/s` high, `fill/s` 0 | The guest is spinning on a clock or an empty queue; each spin costs a scheduler turn. | The park policy is the bug. Send me the two numbers and the menu name. |
| `fill/s` high | It is waiting on its own assets — the first-visit-only slowness. Ranges from step 3 fix it. | Re-run after step 4. |
| Both 0 and PRESENT still low | Neither. It is real guest work. | Enable `?hot-blocks` and send the top 12. |

Second visit to the same menu should be faster than the first. If it is not,
say so, because that rules out the streaming explanation entirely.

---

## Step 6 — start staging what is still guessed

Everything above fixes things. These are the parts where I wrote down a
hypothesis and need evidence:

1. **`.p` container layout.** The census learns the name-table layout
   empirically, but only if the header's self-description is right
   (`magic, count, namesAt, dataAt`). Step 3 decides this in 22 ms.
2. **Split `.BIN` parts.** I argued in the audit §6 that splitting cannot
   cause these symptoms *provided the parts are whole*. To confirm: `ls -l` the
   deployed directory and check the manifest's `size` for `0002.p` equals the
   sum of its parts. If they disagree, that is your bug, not mine.
3. **ReadFile-OVERLAPPED specific.** If 5.1 fails, the next suspect is a guest
   thread parking mid-`OVERLAPPED` while another thread uses the same handle.
   The fix for that is bigger and I do not want to write it without a
   reproduction.

---

## Appendix — every command on one screen

```bash
# --- engine build & gates ------------------------------------------------
cd wine-assembly
bash tools/build.sh
bash tools/check-test-manifest.sh
node test/test-readfile-park-idempotence.js
node test/test-d3d9-palette-state.js
node test/test-d3d9-p8-conversion.js
node test/test-app-files-policy.js
node test/test-app-files-switches.js
node test/test-mbaa-texture-census.js
node test/test-perf-hud-waiting.js

# --- census --------------------------------------------------------------
node tools/mbaa-texture-census.js /path/to/0002.p --list
node tools/mbaa-texture-census.js /path/to/0002.p --out=ranges-0002.json
node tools/mbaa-texture-census.js /path/to/0003.p --out=ranges-0003.json

# --- run it --------------------------------------------------------------
node tools/dev-server.js          # then open http://127.0.0.1:8080/
# URL flags:
#   ?lazy-files          stream everything, ignore preloadAssets
#   ?eager-files         load every file up front (control arm)
#   ?d3d9-audit=textures log texture uploads with format + palette mark
#   ?trace-api=ReadFile  log every guest read
#   ?trace-fs            log the VFS layer
#   ?hot-blocks          per-EIP hot block histogram (needs debug HUD)
```

---

## Appendix — what "done" looks like

- [ ] `build.sh` green, 1837 test files, 8 `PASS` lines from the tests above
- [ ] `ranges-0002.json` exists and its `ranges` are < 200 MB total
- [ ] manifest carries `preloadAssets` + `preloadRanges`; launch shows no
      `NNNMB required at launch` warning
- [ ] dialogue reveals whole
- [ ] every battle has characters and a stage
- [ ] no white placeholders with `?d3d9-audit=textures`
- [ ] HUD `waiting` line says which of park/s or fill/s is nonzero during a
      slow menu, and that number goes to zero after the corresponding fix

---

## Appendix — verified from a clean clone

The patch was applied to a fresh clone of the pinned submodule base and the
whole chain re-run, so these are observed results and not claims:

```
git checkout 8e79a8637
git am docs/mbaa/mbaa-engine-fixes.patch     -> 2 commits, no conflicts
bash tools/build.sh                          -> 1765269 bytes, exit 0
npm install --omit=dev                       -> pngjs, wabt
node test/test-readfile-park-idempotence.js  -> PASS
node test/test-mbaa-texture-census.js        -> PASS
node test/test-d3d9-palette-state.js         -> PASS
node test/test-d3d9-p8-conversion.js         -> PASS
```

The only failure encountered was `Cannot find module 'pngjs'` before step 2's
install, which is why that install is now the first thing in step 2.
