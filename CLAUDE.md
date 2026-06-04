# CLAUDE.md — Freerouting (CHM-T working fork)

Working notes for our fork of [freerouting](https://github.com/freerouting/freerouting),
the Java/Swing Specctra PCB autorouter, used for the CHM-T / pick-and-place workflow.
This file is the operational guide so work can resume on any PC.

## Base version: v1.4.4 (NOT 2.x) — and why

We deliberately base on the **`v1.4.4`** release tag (old `eu.mihosoft.freerouting`
lineage, Java 11), not modern 2.x. This was an evidence-based decision:

| Base | Opening a real KiCad board (dashcam.dsn, 880 segments) |
|------|--------------------------------------------------------|
| master (v2.2.4 +92 commits) | StackOverflow crash, then 180s+ CPU hang. Avoid. |
| v2.2.4 release | Same crash; even with a combine() fix → 180s+ hang |
| **v1.4.4 + our fixes** | **Loads in ~4s, autoroutes correctly** ✓ |

The 2.x `board.normalize_traces` / `PolylineTrace.combine` path is a performance/termination
swamp (it carries `MAX_NORMALIZATION_DEPTH` caps, "infinite loop" band-aids, hard-coded
net-49 debug). The 1.x lineage doesn't have it. Our usual proven-good versions are 1.2.43
and 1.4.4; v1.4.4 is the newest 1.x tag in this repo.

## Build & Run (Java 11)

v1.4.4 needs **Java 11** and Gradle 6.2 (fetched by the wrapper). On this Mac:
```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@11/libexec/openjdk.jdk/Contents/Home
./gradlew jar          # builds build/libs/freerouting.jar (thin jar, no bundled deps)
```

Run the GUI (thin jar needs the runtime deps on the classpath):
```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@11/libexec/openjdk.jdk/Contents/Home
CP="build/libs/freerouting.jar:$(find ~/.gradle/caches/modules-2 \
  -name 'log4j-api-2.13.0.jar' -o -name 'log4j-core-2.13.0.jar' \
  -o -name 'javahelp-2.0.05.jar' | tr '\n' ':')"
java -cp "$CP" eu.mihosoft.freerouting.gui.MainApplication
# add:  -de <design.dsn> -do <out.ses>   to open a design from the CLI
```
Main class: `eu.mihosoft.freerouting.gui.MainApplication`. Logs go to stdout (log4j);
GUI errors appear as Swing dialogs and do NOT hit the log.

## Our fixes on top of v1.4.4

1. **`build.gradle`** — removed three plugins (`net.nemerosa.versioning`, `com.jfrog.bintray`,
   `com.github.ben-manes.versions`) and `jcenter()`; they pulled transitive deps from the
   defunct JCenter so the build failed. Hardcoded the build revision/version they provided.
   Also dropped `apply from: gradle/publishing.gradle` (used the dead bintray plugin).
2. **`designforms/specctra/SpecctraFileScanner.java`** — the generated JFlex lexer rejected
   `{`/`}` ("Illegal character '{'"). KiCad net names use them (`~{MCLR}`, `{slash}`). Fixed
   with a 2-line `ZZ_CMAP` remap of `{`/`}` to the same character class as `~` (accepted).
3. **`designforms/specctra/AutorouteSettings.java`** — the rules reader used the bare
   `AutorouteSettings(int)` constructor, leaving `stop_pass_no` at 0. A `.rules` file with
   `(start_pass_no N)` and no `stop_pass_no` then made the batch autorouter self-stop
   immediately (`start_pass_no > stop_pass_no` at `BatchAutorouter.autoroute_passes()`),
   routing nothing. Now defaults `start_pass_no=1`, `stop_pass_no=MAX_VALUE` after construction.
4. **`interactive/BatchAutorouterThread.java`** — added one INFO line logging the
   incomplete-connection count + autoroute settings at the start of a batch run (handy telemetry).

### Gotchas worth remembering
- Opening a `.dsn` with a sibling `.rules` shows a "Please confirm importing stored rules"
  popup. "Yes" loads the stored rules (incl. a persisted `start_pass_no` — see fix #3);
  "No" uses DSN defaults.
- In the GUI, while the autorouter runs the board is read-only and a **left-click on the
  board stops it** (by design): `BoardHandling.left_button_clicked` → `request_stop()`.
- The thin jar is not runnable via `java -jar` (no Main-Class / no bundled deps) — use the
  classpath form above.

## Features added in this fork

- **Stop-at-minimum autorouting (revert-to-best)** — *done*. The batch autorouter records the
  board state of the completed pass with the fewest incomplete connections (`BatchAutorouter`
  `min_incomplete_count`/`best_board`, via a per-pass `RatsNest` count + `RoutingBoard.clone()`).
  When the run stops (manual background-click or natural end), `restore_best_board_if_better()`
  reverts the live board to that best pass through `BoardHandling.restore_routing_board_after_autoroute()`
  and shows the ratsnest, so you land on the most-complete result for manual finishing. Clicking
  to stop also puts an immediate "Stopping autorouter…" acknowledgement on the status bar.
  Known limitation: stopping during the very first pass (no completed pass yet) leaves the
  partial first-pass board (nothing better to revert to).
- **"Find Incompletes" highlight** — *done*. A toolbar button (next to Autorouter) calls
  `BoardHandling.highlight_incompletes()`, which recomputes the ratsnest and blinks every
  incomplete a few times (~3s) as a bold yellow airline with a **fixed-screen-size ring** at
  each endpoint (`draw_incomplete_highlight()`), so even very short incompletes are findable at
  any zoom. The status bar reports the count. Auto-clears via a Swing `Timer`.

## Beautify engine (multi-rule cleanup) — spec from user before/after examples

Goal: a post-route "Beautify" pass (toolbar button, on-demand, re-runnable) that cleans up routed
geometry to these rules. Built rule-by-rule; hardest last. (Branch: `beautify`.)

- **Rule 1 — straight pad exits** *(REUSE APPROACH RULED OUT — see findings; needs a from-scratch
  geometry approach)*. A trace must leave its pad with a **straight, perpendicular stub** (0.010")
  out the face it heads toward (e.g. the short/"south" face of an elongated SMD pad), *then* turn at
  45° — never exit at 45° straight off the pad corner.

  **Overnight findings (2026-06-03, blind/headless — must re-verify with eyes on a real bad exit):**
  Tried reusing `PolylineTrace.correct_connection_to_pin()` (raise `pin_edge_to_turn_dist` to 0.010"
  = 2540 board units, board unit = 0.1µm — conversion verified correct — then run it over all pad
  ends). **This is a no-op on freerouting-routed boards** and was reverted. Headless test on
  dashcam (autoroute 1 pass, and autoroute+postroute): of ~709 pad-connected trace ends,
  **~706 already PASS** `check_connection_to_pin()` even at the 0.010" requirement, and the ~3 that
  fail can't be corrected (clearance) and are skipped → **0 changed**. The autorouter/optimizer
  already produce perpendicular exits with long-enough straight segments *by freerouting's own
  definition*. This matches the user's live test (clicking Beautify gave "0 beautified").
  CONCLUSION: freerouting's legality check (`check_connection_to_pin`) says the user's exits are
  already fine, yet the user *sees* 45° exits → the discrepancy must be diagnosed directly.
  **Next step (do FIRST, with the user): pick one visibly-bad exit, log that exact trace's corner
  coordinates + last-segment direction/length + `check_connection_to_pin` result**, to learn what
  freerouting sees vs what the user sees. Only then design the real fix (likely a from-scratch
  reconstruction of the exit polyline that forces a 0.010" perpendicular stub + 45° transition,
  independent of freerouting's "is it legal" check; possibly the bad exits are from *manual* routing
  which doesn't enforce exit restrictions). The reverted prototype lives in git history on this
  branch if useful as scaffolding.
- **Rule 2 — via entry/exit at 45°/90°, centered**. Traces must meet a **via center** at 45/90
  increments, not "touching any way possible." Extend the exit-restriction idea to vias (drill items).
- **Rule 3 — no acid traps**. Eliminate acute-angle junctions where traces meet pads/traces.
  freerouting has some acid-trap removal already; strengthen it.
- **Rule 4 — spacing & channels** *(hardest; likely partial)*. Maximize spacing where there's room,
  even/consistent spacing, and route through the **central channel between pads** instead of hugging
  them. Per-segment lateral shift via `Line.translate` (IntPoint-only → 45/90 boards) validated
  against the search tree; "route through channel instead of next to pad" may need re-routing.

## Git

- `origin` → upstream `freerouting/freerouting` (modern 2.x; for reference only).
- `c-riegel` → our fork `github.com/c-riegel/freerouting`.
- Working branch: **`v1.4.4-1`** (this branch) = `v1.4.4` + the four fixes above.
