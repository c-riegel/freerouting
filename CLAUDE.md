# CLAUDE.md — Freerouting Working Notes

Working notes for our fork of [freerouting](https://github.com/freerouting/freerouting)
(a Java/Swing Specctra PCB autorouter). This file captures local build/run intel and
the active bug we're chasing so work can resume on any PC.

> The upstream project ships a large `AGENTS.md` with deep architecture notes — read it
> for subsystem detail. This file is *our* operational layer on top of it.

## Build & Run

- **Requires Java 25.** The system default `java` may be older (e.g. 11). On this Mac,
  JDK 25 is installed via Homebrew (`openjdk@25`). Always point `JAVA_HOME` at it:
  ```bash
  export JAVA_HOME=/opt/homebrew/opt/openjdk@25/libexec/openjdk.jdk/Contents/Home
  ```
  (On other PCs, set `JAVA_HOME` to wherever JDK 25 lives. `build.gradle` pins
  `JavaLanguageVersion.of(25)`.)
- **Build:** `./gradlew build -x test`  → artifacts in `build/libs/`.
- **Run the GUI:** `./gradlew run`
- **Run headless / load a design from CLI:**
  ```bash
  ./gradlew run --args="-de path/to/design.dsn -do /tmp/out.ses -dl off"
  ```
  - `-de <file>` design input (DSN/SES/RULES), `-do <file>` output SES,
    `-di <dir>` input dir, `-dl off` disables some logging. Arg parsing lives in
    `src/main/java/app/freerouting/settings/GlobalSettings.java` (~line 471).

### Gotchas
- `build/libs/freerouting.jar` is **NOT** a runnable fat jar — no `Main-Class`, no
  bundled deps. `java -jar freerouting.jar` fails with "no main manifest attribute".
  Use `./gradlew run` (or build a fat jar if we want a standalone artifact).
- `./gradlew tasks` currently **fails** under Gradle 9: a custom `installLocalGitHook`
  task calls `Copy.fileMode(int)`, which was removed in Gradle 9
  (`No signature of method ... Copy.fileMode()`). Does **not** affect `build` or `run`.
  Fix later by switching to `filePermissions { unix(...) }`.
- The app phones home to `api.freerouting.app` for analytics/version checks; harmless
  `501` warnings appear in logs when offline.

## Active Bug — StackOverflowError opening `dashcam.dsn`

**Symptom:** Opening `~/Documents/git/dashcam/dashcam.dsn` (KiCad 9.0.7 export, um units)
throws `java.lang.StackOverflowError` during board load.

**Root cause (recursion):** `app.freerouting.board.PolylineTrace.combine()`
(`PolylineTrace.java:170`) is tail-recursive — when `combine_at_start`/`combine_at_end`
report "something changed", it calls `this.combine()` again to keep merging. For this
board it recurses unboundedly (observed 40+ frames of `combine()` at line 180 before the
deeper geometry stack blew). The inner work that actually overflows is in the geometry
layer:
```
PolylineTrace.combine (PolylineTrace.java:180)  ← recurses
  → combine_at_end → SearchTreeManager.insert → ShapeTree.insert
    → Item.tree_shape_count → PolylineTrace.calculate_tree_shapes
      → ShapeSearchTree.offset_shape → Polyline.offset_shapes
        → TileShape.intersection_with_simplify → IntOctagon.intersection
          → Simplex.intersection (recurses, Simplex.java:631/686)
            → Simplex.remove_redundant_lines (Simplex.java:886)
              → Line.fast_equals (Line.java:97)   ← top of trace
```

**Hypotheses to investigate (not yet confirmed):**
1. `combine()` oscillates — two traces (or a trace with itself) repeatedly report a
   successful combine without geometrically converging, so the tail recursion never
   bottoms out. Converting the tail recursion to a loop would only turn the crash into a
   hang, so the real fix is detecting the no-progress / degenerate-geometry case.
2. Degenerate/zero-length or duplicate-corner trace geometry from the KiCad export feeds
   `Simplex.remove_redundant_lines` a case it doesn't reduce.

**Repro (headless, fast):**
```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@25/libexec/openjdk.jdk/Contents/Home
./gradlew run --args="-de /Users/chris/Documents/git/dashcam/dashcam.dsn -do /tmp/dashcam.ses -dl off"
```
The `dashcam.dsn` file is in a sibling repo (`../dashcam`), not in this repo.

**Key files for the fix:**
- `src/main/java/app/freerouting/board/PolylineTrace.java` (`combine`, `combine_at_start`,
  `combine_at_end`)
- `src/main/java/app/freerouting/geometry/planar/Simplex.java`
  (`remove_redundant_lines`, `intersection`)
- `src/main/java/app/freerouting/geometry/planar/Line.java` (`fast_equals`)

## Git / GitHub

- **Upstream `origin`:** `https://github.com/freerouting/freerouting.git`
- **Our fork:** push to the `c-riegel` GitHub account (logged in via `gh`). Plan: add our
  fork as a remote (e.g. `c-riegel`) and push the working branch there; keep `origin`
  pointing at upstream for pulling fixes.
