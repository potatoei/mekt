# HANDOVER — read this before touching anything

Written 2026-11-17 for the next agent picking up this project. The agent before
you (me) made a series of mistakes that cost the project owner real work. Those
mistakes and their causes are documented here so they are not repeated. Read
§1 and §7 first; they are the ones that matter.

---

## 1. Who owns what, and why it is confusing

There are **three** repositories in play. Confusing them is what caused most of
the damage.

| Repository | Owner | Role |
|---|---|---|
| `potatoei/mekt` | the project owner | the **root** repo. Only `docs/`, `net/`, `wine-assembly/` (submodule), `README.md`, `.gitignore`, `.gitmodules`. **No `site/`, no assets, no game data.** |
| `stevenvinhtran/wine-assembly` | a third party | the **upstream engine**. The owner cannot push here (`403 Permission denied`). |
| `potatoei/wine-assembly` | the project owner | a **fork** of the above. Exists so the engine has somewhere the owner *can* push. |

### The rule that follows from this

**Never point a submodule at a commit that exists only in your sandbox.**
A submodule pointer is a 40-character promise that the commit is fetchable. If
it is not, every clone of the repo fails with:

```
fatal: remote error: upload-pack: not our ref <sha>
```

This happened. The branch pointed at `7cd76efa`, which existed only inside the
sandbox. It was later reverted to `c3f0474`, which exists on **both** the fork
and upstream (verified by fetching it into an empty repo from each). **Current
submodule pointer is `c3f0474` and that is correct.**

Verify before you change it:

```bash
# from an EMPTY repo, so a local object cache cannot give a false positive
git init /tmp/probe && cd /tmp/probe
git remote add o https://github.com/potatoei/wine-assembly.git
git fetch --depth=1 o <sha>        # fails loudly if the sha is not reachable
```

---

## 2. Current state of the branch

- Branch: `arena/42e272c4-mekt`, PR **`potatoei/mekt#1`**
- Submodule `wine-assembly` → `c3f0474` (the owner's fork's `main`)
- The engine fixes are **not** in the submodule. They ship as a patch:
  `docs/mbaa/mbaa-engine-fixes.patch`, **baselined on `c3f0474`**, applies with
  zero conflicts.
- The owner's `site/` directory and game assets are **not in any repo**. They
  were never committed. See §7.

### The two documents that already exist

- `docs/mbaa/AUDIT_2026-11-17.md` — the five reported issues, each with a
  mechanism, a proposed fix, and what measurement would settle it.
- `docs/mbaa/RUNBOOK.md` — ordered steps to apply and verify the fixes.

Read both before proposing anything. This file only adds what they omit: the
history of what went wrong, and the traps.

---

## 3. The project, in one paragraph

A browser-native WebAssembly port of *Melty Blood Actress Again Current Code*
(MBAACC v1.07). A Windows x86 PE binary runs on an x86-to-WASM compatibility
layer (`wine-assembly`) that implements Win32, Direct3D 9 and DirectSound on
top of WebGL/WebAudio. Game data lives in `.p` container archives
(`0002.p` sprites ~900 MB, `0003.p` stages ~140 MB, `0005.p` audio ~326 MB).
Those archives are far over the 16 MiB eager threshold in
`lib/app-files.js`, so they stream over HTTP ranges and the guest **parks** on
a cache miss.

---

## 4. The five reported issues and what was found

| # | Report | Cause found | Fixed? |
|---|---|---|---|
| 1 | Stream all textures before play | archives stream lazily; a failed fill is cached as NULL by the guest forever | policy implemented, needs the owner's sizes |
| 2 | Battles with no characters, black stage | same family as #1 — the NULL cache | with #1 |
| 3 | Dialogue shows "The" → "Th" → "The..." | **`ReadFile` wrote the byte count only to `OVERLAPPED.InternalHigh`, never to `*lpBytesRead`; and wrote a count on every *parked* attempt** | yes, tested |
| 4 | White/grey placeholder textures | **`D3DFMT_P8` uploaded the palette INDEX as greyscale**; no palette was ever stored | yes, tested |
| 5 | Menus 20 FPS, battle 60 | not proven. Two candidates: clock-spin park cost, or lazy fills. HUD counters added to tell them apart | instrumented, needs a screenshot |

### The three defects that were not in any report

1. **Park contract** (`lib/filesystem.js`, `src/09a7d-handlers-shell-file.wat`).
   A park is re-entered from the top, so writing `0` to `*lpBytesRead` is not
   "zero bytes" — it is "this call has not happened yet". The guest samples it
   and publishes a truncated count. Test:
   `test/test-readfile-park-idempotence.js`.
2. **Launch rejected its own preload shape** (`host.js` ~line 3949). It threw on
   `loadMode:'lazy'` + `preloadRanges`, which is exactly what `lib/app-files.js`
   emits. The feature could never have worked.
3. **P8 greyscale** (`lib/d3d9-host.js`). Also present in `c3f0474`. Tests:
   `test/test-d3d9-palette-state.js`, `test/test-d3d9-p8-conversion.js`.

---

## 5. The address collision — read before rebasing again

The fixes were originally written against submodule base `8e79a863`, which is
**42 commits behind** the fork's `main` (`c3f0474`). Rebasing collided on 10
files. Most were mechanical; one was not:

**`src/09ad-handlers-d3d9.wat`, device-state offsets.**

| Offset | What it is in `c3f0474` |
|---|---|
| `+25616/+25620/+25624` | **render-target slots 1–3** (`$d3d9_color_binding_extra`) |
| `+25628` | pixel constants c8–c31 |
| `+26012` | end of the historical block |

The palette work had placed the 16 palettes at `+25620`, which in `c3f0474` is
a render-target slot that `lib/d3d9-host.js` reads. Overlaying them would have
corrupted every render-target bind — a silent bug that surfaces as garbage in
battle much later. **The palettes were moved after the historical block:**

```
+26012            current palette number
+26016 + 1024*n   16 palettes (256 D3DCOLORs each), through +42399
d3d9_program_alloc: 26012 -> 42400
```

The accessors `$d3d9_palette_ptr`, `SetCurrentTexturePalette`,
`GetCurrentTexturePalette` and `lib/d3d9-host.js`'s `paletteOf` all use these
offsets. **If either side moves, `test/test-d3d9-palette-state.js` is the thing
that says so.**

Also preserved on rebase: the fork's texture `identity` cache key (a mip record
reused by a new texture must never match) *alongside* the palette mark, and the
fork's `audioThreadStepMs` / `fineQpc` / `d3d9FullNpot` / `d3d9PS20` / native
hook slots *alongside* `preloadAssets`.

---

## 6. Verification — what green looks like

```bash
cd wine-assembly
npm install --omit=dev        # pngjs + wabt; without it the d3d9 tests die
                              # with "Cannot find module 'pngjs'" and it looks
                              # like a broken change. It is not.
bash tools/build.sh           # 1769509 bytes, region hash 5d0cb5d37037987b
node tools/check-silent-stubs.js   # pinned: 243 manual + 143 metadata
bash tools/check-test-manifest.sh  # 1837 files
```

Tests that must print `PASS`:

```
test/test-readfile-park-idempotence.js
test/test-d3d9-palette-state.js
test/test-d3d9-p8-conversion.js
test/test-app-files-policy.js
test/test-app-files-switches.js
test/test-mbaa-texture-census.js
test/test-perf-hud-waiting.js
```

**Pre-existing failures, not caused by these changes** (verified identical on a
clean tree): `test-d3d9-adapter.js` (`vs_1_1`),
`test-d3d9-setup-scheduler.js`, every `*-web.js` (needs puppeteer), and the
`calc.exe`-dependent d3d9 tests.

### Tooling added

- `tools/mbaa-texture-census.js` — emits `preloadRanges` JSON for only the
  texture members of a `.p` archive. Pages the file (42 MB RSS / 22 ms for a
  1.6 GB archive, measured). **Refuses and writes nothing when it cannot parse
  the container** — a wrong range list pins the wrong bytes, which is worse
  than pinning nothing.
- Perf HUD `waiting N park/s M fill/s` — parks = guest spinning on a clock that
  has not ticked; fills = blocked on a lazy chunk. Which is nonzero names the
  bug. Red when nonzero.

---

## 7. What went wrong, so you do not repeat it

### 7.1 The owner's work was destroyed

`site/` — a directory of the owner's own modifications — was deleted along with
`site/chunks/`. Those chunks held **reassembled game data** the owner had
rebuilt by hand.

Facts, verified:

- `site/` **never existed** in `potatoei/mekt` or `stevenvinhtran/melty-web`.
  `git log --all -- 'site/*'` is empty in both.
- `git checkout`, `git fetch` and `git submodule update` **do not delete
  untracked or ignored files.**
- The operations that *do*: `git clean -xfd`, `git clean -fd`,
  `git submodule deinit`, `git submodule rm`, or re-cloning a submodule
  directory (which empties it first).

**Therefore: before suggesting any command that can touch the working tree,
ask the owner whether they have uncommitted or untracked work, and where.**

The agent (me) never asked. It handed over `git submodule update --init` — which
is destructive to untracked files *inside the submodule* — without asking.

### 7.2 The agent assumed instead of asking

Repeatedly. The worst instances:

- Wrote `/path/to/mekt` as a placeholder, and the owner reasonably read it as
  "a path I should already have".
- Told the owner to change `.gitignore` to protect game files. **That is
  wrong**: ignored files are invisible to git, so git neither tracks nor
  protects them. Changing ignore rules cannot prevent deletion.
- Kept adding commits on top of a broken submodule pointer instead of fixing
  the pointer first.
- Proposed `tar --exclude=...` for a backup the owner had said should be
  "entire folders". Do not silently narrow what the owner asked for.

### 7.3 The archives are NOT bigger than the tree (the 3.77 GB scare)

`tar -czf` on incompressible data grows by ~0.03%, never 40%. A 2.68 GB tree
producing a 3.77 GB archive means the archive contains **more than the tree**.
Two causes, both common:

1. `tar -czf out.tgz .` **packs an existing `out.tgz` into itself.**
2. `.git` alone can be gigabytes.

Safe shape — write outside the tree and compare byte counts:

```bash
cd ~/melty-web
tar -czf /tmp/ctx.tgz --one-file-system .
echo "tree:    $(du -sb . | cut -f1)"
echo "archive: $(stat -c%s /tmp/ctx.tgz)"
tar -tzf /tmp/ctx.tgz | head -20
```

`du -sb` gives raw bytes, so the numbers are directly comparable.

---

## 8. Traps for the next agent

1. **`site/` is the owner's. It is not in any repo. Never assume it is
   disposable.**
2. **Do not run `git clean`, `submodule deinit`, or `submodule update --init`
   on the owner's machine without asking first, in those words.**
3. **Never push a submodule pointer to an unpushed commit.** Verify the sha is
   fetchable from a *clean* repo (§1).
4. Reading a sha from a repo where it already exists is a **false positive**.
   Always probe from an empty repo.
5. `createFile(..., 1)` is `CREATE_NEW` and fails on an existing file; the VFS
   tests use `3` (`OPEN_EXISTING`).
6. `RegionMap.g2w(guest, imageBase)` is how the host translates a guest address;
   reading the raw linear offset gives someone else's page.
7. `fs_read_file` is the **legacy BOOL** wrapper (`1` = success, `0` = look at
   `fs_read_pending`) and is *not* an error code. `fs_read_file_at` is the
   Win32-shaped one (`0` ok, `997` park, else Win32 error).
8. `patternRegex` anchors the **full** VFS path; a repo-relative
   `assets/raw/...` entry can never match it — that is why `lib/app-files.js`
   has a tail-anchor branch.
9. The "eager: small app" shortcut fires before preload logic when the total is
   under `APP_EAGER_BYTES` (16 MiB). Test fixtures must exceed it.
10. `docs/dos-corpus/` is 105 MB of unrelated DOS test programs. Scope greps to
    `lib/ src/ tools/` or the output is unusable.
11. Skim `docs/mbaa/RUNBOOK.md` §"Troubleshooting" before answering a
    checkout-conflict question. `README.md` is byte-identical between `main`
    and the branch (blob `0d0fad6`); a conflict about it is always about the
    caller's working tree.
12. The owner is on **WSL/Windows**. `core.autocrlf` causes phantom "modified"
    files with empty diffs. A root `.gitattributes` now pins `eol=lf`.

---

## 9. What the owner still needs to provide

1. A `.p` census (`node tools/mbaa-texture-census.js 0002.p --out=r.json`) —
   the difference between a 1.04 GB launch and a few tens of MB.
   Until then `preloadAssets` makes `mbaa` wait for 1040 MB before first frame;
   the policy warns above 64 MB and `?lazy-files` bypasses it per launch.
2. A HUD screenshot during a 20 FPS menu.
3. A `?d3d9-audit=textures` log from a battle with white placeholders.
4. Whether the real `site/` layout differs from anything committed.

### GitHub size limits, if the owner backs the project up

- **>100 MiB per file: push rejected (GH001).** Not configurable.
- **>50 MiB per file: warning, push succeeds.**
- Browser upload: 25 MiB per file.
- Repo: stay under 1 GB ideally, 5 GB is the practical ceiling.
- Single push over 2 GB: rejected.

`0002.p` is ~900 MB. **It cannot go into a normal git repo.** Use Releases
(2 GiB per file), Cloudflare R2, or Git LFS (2 GB/file on Free, 10 GiB/month
bandwidth). Never assume a data file is committed just because it is present on
disk.
