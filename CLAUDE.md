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

## Planned features (next)

- **Stop-at-minimum autorouting**: stop automatically at the pass with the fewest unconnected
  traces and don't start another round (today you stop/start manually to chase the minimum).
- **Post-optimization beautify pass**: center trace exits on pad edges (no off-angle stubs);
  distribute parallel traces evenly for maximum spacing / minimal crosstalk.

## Git

- `origin` → upstream `freerouting/freerouting` (modern 2.x; for reference only).
- `c-riegel` → our fork `github.com/c-riegel/freerouting`.
- Working branch: **`v1.4.4-1`** (this branch) = `v1.4.4` + the four fixes above.
