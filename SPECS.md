# COBOL-on-JVM Bridge — Windows-Native Environment & Milestone Roadmap

**Architecture (unchanged):**
COBOL source → GnuCOBOL `cobc -m` → `.dll` module → Java FFM bridge (`java.lang.foreign`) → `-javaagent` + bytecode transformer → "ordinary-looking" Java service

All commands are PowerShell on native Windows 10/11 x64. No WSL, no SDKMAN.

**How this document is organized**

- **Part 1 — Environment setup** (every team member does this on their own machine)
- **Section A — Primary roadmap:** a chain of milestones, each with an exit criterion instead of a date
- **Section B — Independent workstreams:** the same work split among three people, built around agreed interfaces so nobody blocks anyone

---

## Part 1 — Dev Environment Setup

### Ground rule: everything is 64-bit

The JVM can only load DLLs of its own bitness. Use an **x64 JDK** and an **x64 GnuCOBOL build** (MinGW-w64 or MSVC x64). A 32-bit GnuCOBOL is the most common cause of DLL load failures.

### JDK: Eclipse Temurin 25 (LTS)

Temurin is the free, TCK-certified OpenJDK build. Version 25 is the current LTS, and the Foreign Function & Memory API has been a final, non-preview feature since JDK 22, so the bridge targets a stable API.

```powershell
winget install EclipseAdoptium.Temurin.25.JDK
# open a NEW terminal so PATH/JAVA_HOME refresh, then:
java -version     # should report Temurin 25
echo $env:JAVA_HOME
```

If `JAVA_HOME` isn't set by the installer (use the actual folder name under `C:\Program Files\Eclipse Adoptium\`):

```powershell
[Environment]::SetEnvironmentVariable("JAVA_HOME", "C:\Program Files\Eclipse Adoptium\jdk-25.x.y-hotspot", "User")
```

### Build tool: Gradle (via the wrapper)

Gradle's task graph handles the non-standard step (run `cobc` to produce the DLL before Java needs it). Commit the **wrapper** so nobody depends on a system-wide Gradle.

```powershell
winget install Gradle.Gradle        # only to run `gradle init` once
gradle init --type java-application
.\gradlew.bat --version             # confirms the JDK 25 toolchain is picked up
```

### GnuCOBOL (the COBOL compiler)

There's no official one-click Windows installer. Pick **one** route and have the whole team use the same one:

| Route | Notes |
|---|---|
| **Prebuilt Windows binaries** (3.2, x64) | Fastest. Download a 3.2 x64 build from the GnuCOBOL project's Windows binary links and unzip to e.g. `C:\gnucobol`. |
| **MSYS2 / MinGW-w64** | Check the package exists first: `pacman -Ss gnucobol` in a UCRT64 shell. |

Use the **stable 3.x** line, not the 4.0 early-release branch.

Put its `bin` folder on `PATH`. This matters twice: for `cobc` itself and for the `libcob-*.dll` that your compiled module needs at load time.

```powershell
$cob = "C:\gnucobol\bin"      # adjust to your install
[Environment]::SetEnvironmentVariable("Path", "$env:Path;$cob", "User")
```

If your build is MSVC-based, `cobc` also needs the Visual Studio compiler environment: run builds from an **x64 Native Tools** prompt or have Gradle call `vcvars64.bat` first. A MinGW-based build avoids this.

Verify it produces a loadable module:

```powershell
cobc --version
"       IDENTIFICATION DIVISION.`n       PROGRAM-ID. HELLO.`n       PROCEDURE DIVISION.`n           DISPLAY 'HELLO'.`n           GOBACK." | Set-Content hello.cbl
cobc -m -o hello.dll hello.cbl
```

If `hello.dll` appears, the toolchain works. Note which `libcob-*.dll` ships in your `bin` folder; you'll need that name for the bridge.

### Bytecode library

A Gradle dependency, not a system install. **ASM 9.7 predates JDK 25 class files**, so use the newest 9.x release (check Maven Central) for `asm` and `asm-commons`. The no-dependency alternative is the JDK's built-in **ClassFile API** (final in JDK 24). Decide once, in Milestone 3, and stick with it.

### Gradle configuration

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

// Build the COBOL module before anything that needs it
tasks.register('buildCobol', Exec) {
    inputs.dir 'src/main/cobol'
    outputs.dir layout.buildDirectory.dir('cobol')
    doFirst { layout.buildDirectory.dir('cobol').get().asFile.mkdirs() }
    commandLine 'cobc', '-m', '-o', 'build/cobol/calcint.dll', 'src/main/cobol/calcint.cbl'
}

tasks.withType(JavaExec).configureEach {
    dependsOn 'buildCobol'
    jvmArgs '--enable-native-access=ALL-UNNAMED'
}

tasks.test {
    dependsOn 'buildCobol'
    jvmArgs '--enable-native-access=ALL-UNNAMED'
}
```

Once the agent exists, add `-javaagent:build\libs\agent.jar` (the agent JAR manifest needs `Premain-Class` and `Can-Retransform-Classes: true`).

### Sanity check

```powershell
java -version          # Temurin 25.x, x64
.\gradlew.bat --version
cobc --version         # GnuCOBOL 3.x
```

### Windows pitfalls to know up front

- **Locked DLLs.** A DLL loaded by a running JVM can't be overwritten. If a rebuild fails on `calcint.dll`, a stale `java.exe` or the Gradle daemon still holds it. Run `.\gradlew.bat --stop` and kill leftover JVMs.
- **DLL dependency search.** Loading the module also needs `libcob-*.dll` and its dependencies resolvable. Keep the GnuCOBOL `bin` on `PATH`, or load `libcob` explicitly first.
- **Antivirus / Defender.** Freshly built unsigned DLLs can be scanned or quarantined mid-build. Exclude the project's `build\` folder if you see random access-denied errors.
- **Paths.** Use `Path`/`Paths` in Java, quote paths in Gradle, and avoid spaces in the GnuCOBOL install path. Keep the repo near the drive root (e.g. `C:\dev\cobol-bridge`) to stay clear of the 260-character limit.
- **Native stdout.** COBOL `DISPLAY` writes through the C runtime, not `System.out`, so it won't appear in Java-captured output or test reports. Return values through arguments instead.

---

# Section A — Primary Roadmap (Milestones)

Milestones are ordered by dependency. A milestone is done only when its **exit criterion** is met, not when time has passed. Two external gates (the **screening review** and the **final showcase**) sit in the chain; everything before a gate must be done to enter it.

### M0 — Environment ready
- Every team member passes the three sanity checks and builds the empty Gradle project
- Same GnuCOBOL route and version used by everyone
- **Exit:** `hello.dll` builds from `.\gradlew.bat buildCobol` on each machine

### M1 — Call boundary proven
- Trivial COBOL program compiled with `cobc -m`
- Java program using `java.lang.foreign` (`Linker`, `SymbolLookup`, `MethodHandle`) loads the DLL and calls the entry point
- libcob initialized correctly (`cob_init` before the module entry point; arguments passed by reference as raw pointers)
- `--enable-native-access=ALL-UNNAMED` in all run configs
- **Exit:** a Java test calls COBOL and asserts on a returned value (not on `DISPLAY` output)

### M2 — Packed decimal is trustworthy
- COBOL program with a `COMP-3` field and a simple arithmetic op
- `MemorySegment`-based packed-decimal encoder/decoder
- Round-trip unit tests, bit-for-bit
- At least one `ROUNDED` / truncation test, so correctness covers arithmetic and not just encoding
- Byte layout documented
- **Exit:** the decimal test suite is green and every case's expected value came from running real COBOL, not from the Java side's own assumptions
- *Highest-risk milestone. Do not start M3 integration until this is solid.*

### M3 — Bytecode swap works end to end
- `java.lang.instrument` agent (`premain`, manifest attributes, `-javaagent` wiring)
- Transformer detects `@CobolBacked("CALCINT")` and rewrites the method body to invoke the `MethodHandle`
- One real COBOL program (interest calculator or similar) behind it
- Transformer writes rewritten class bytes to `build\transformed\`; `javap -c` runs on those files (plain `javap` on the build output only shows the original)
- ASM-vs-ClassFile-API decision made and recorded
- **Exit:** a Java method with an empty/unimplemented body returns the correct COBOL-computed value, and the before/after `javap` dumps are saved

### M4 — Breadth: state and control flow
- Second COBOL program with **persistent WORKING-STORAGE** via a "context handle" the Java side holds and passes back
- Third program with loop/conditional logic
- Concurrency model: **start with a single lock (or one dedicated COBOL thread)**, since libcob keeps global runtime state; per-thread contexts are a stretch goal only
- **Exit:** three COBOL-backed methods work through the agent, and a concurrent-calls test shows no state corruption

### M5 — Demo service shell
- Plain Java service class (no framework) exposing the three COBOL-backed methods like normal Java
- Integration test suite covering all three programs
- Demo launcher script `demo.ps1` (sets `PATH`, builds, runs with the right JVM flags)
- **Exit:** `demo.ps1` works from a clean shell with no manual setup

### M6 — Screening package ready (gate: screening review)
- Live demo scripted or recorded: ordinary source → run → `javap` dump proving COBOL ran underneath
- One-page plain-language explainer for non-engineers
- Architecture diagram
- Security-question answer ready: opt-in via explicit annotation; same class of mechanism as APM/profiling agents already trusted in production
- 60-second pitch and 5-minute walkthrough rehearsed separately
- **Exit:** someone outside the team can follow the pitch, and the demo runs from a clean checkout

### M7 — Screening feedback triaged
- Every question the team couldn't answer cleanly is written down
- **Exit:** the list is ranked and becomes the input to M8, replacing any generic backlog

### M8 — Control-flow and decimal hardening
- Expand supported COBOL constructs based on what real sample programs need (arrays, more arithmetic ops)
- Dedicated rounding, truncation and size-error tests
- **Exit:** the top items from M7 are resolved, and the decimal edge-case suite passes

### M9 — Data story and polish
- Benchmark: FFI-bridged call vs. native COBOL vs. (if useful) naive JNI, reduced to one clean number
- README (including the Windows setup from Part 1), comments, consistent structure
- Short write-up of the approach for recruiters to forward
- **Exit:** a stranger can clone the repo and run the demo from the README alone

### M10 — Feature freeze and rehearsal
- No new features after this point
- Backup demo video recorded
- Feedback from at least one person unfamiliar with the project; pitch revised if they couldn't follow it
- **Exit:** frozen branch tagged, backup video exists, pitch passes the unfamiliar-listener test

### M11 — Showcase ready (gate: final showcase)
- Final rehearsal on the exact machine used for the live demo
- Live demo and backup video both on hand
- One-pager printed or on a tablet
- **Exit:** full run-through completed without touching the code

---

# Section B — Independent Workstreams (3 People)

Three people, three workstreams, each with its own deliverables and a stub of the others' pieces so nobody waits. Everyone does Part 1 and M0 first.

## Shared contracts (agree on these first, then don't change them without telling the others)

Keep these in a `contracts/` folder in the repo.

1. **Annotation:** `@CobolBacked("PROGRAMNAME")` on a method, retained at runtime.
2. **Bridge interface (Java):** a single entry point, e.g. `CobolBridge.call(String program, MemorySegment args)` returning an integer status. Person B codes against this interface only.
3. **Program catalog:** for each COBOL program, name, argument layout (field types, `COMP-3` precision/scale, byte offsets), and expected results for sample inputs. Initial three: `CALCINT` (stateless interest calculation), `ACCTSTATE` (persistent WORKING-STORAGE with a context handle), `LOOPCALC` (loop/conditional).
4. **Test-vector file:** known inputs and expected outputs per program, produced from real COBOL runs. All three people test against this.

## Person A — Native bridge owner
*Owns: everything from COBOL source to a working `MethodHandle`.*

| Task | Needs from others | Done when |
|---|---|---|
| Standardize the GnuCOBOL install route and the `buildCobol` Gradle task | Nothing | Team builds `hello.dll` identically |
| Write the three COBOL programs from the catalog | Catalog layouts | Each compiles to a `.dll` and runs standalone |
| FFM loader: `Linker`, `SymbolLookup`, libcob init and DLL dependency handling | Nothing | M1 exit criterion met |
| Packed-decimal encoder/decoder + round-trip and rounding tests | Test-vector format | M2 exit criterion met |
| Context-handle design for `ACCTSTATE` | Catalog | Stateful program works across calls |
| Concurrency: single lock or dedicated COBOL thread, plus the concurrent-calls test | Nothing | No state corruption under test |
| Implement `CobolBridge` for real | Interface from contracts | B's agent works with the real bridge swapped in for the stub |

**Independence:** produces and tests everything with plain Java test code. Never needs the agent.

## Person B — Agent and transformer owner
*Owns: the bytecode swap.*

| Task | Needs from others | Done when |
|---|---|---|
| Agent scaffold: `premain`, manifest, `-javaagent` wiring | Nothing | Agent loads and logs |
| ASM vs. ClassFile API decision (newest ASM 9.x or built-in) | Nothing | Decision recorded |
| Transformer: detect `@CobolBacked`, rewrite the method body to call `CobolBridge` | Annotation + interface | Rewritten method calls the bridge |
| Dump transformed bytes to `build\transformed\` and capture `javap -c` before/after | Nothing | Dumps saved |
| Transformer edge cases: loops, conditionals, multiple annotated methods, argument marshalling into `MemorySegment` | Argument layouts from the catalog | M3 and M4 exit criteria met |
| Swap the stub bridge for A's real one | A's `CobolBridge` | End-to-end call returns real COBOL result |

**Independence:** develops against a **fake `CobolBridge`** that returns canned values from the test-vector file. The transformer is verifiable without a single COBOL compile.

## Person C — Service, verification and communication owner
*Owns: the demo, the evidence, and the story.*

| Task | Needs from others | Done when |
|---|---|---|
| Repo scaffold, Gradle wrapper, CI-style `.\gradlew.bat test` entry point | Nothing | Clean checkout builds |
| Plain Java service class with three `@CobolBacked` methods (empty bodies) | Annotation + catalog signatures | Compiles; behavior arrives when A and B land |
| Integration test suite driven by the shared test-vector file | Test vectors | Fails against the stubs, passes once A and B integrate |
| `demo.ps1` launcher (PATH, build, JVM flags) | Nothing | Runs from a clean shell |
| Benchmark harness: bridged vs. native COBOL vs. optional JNI | A's bridge | One clean number |
| One-pager, architecture diagram, security answer, 60-second pitch | Nothing | M6 exit criterion met |
| README, write-up, backup demo video, unfamiliar-listener test | Working demo | M9 and M10 exit criteria met |

**Independence:** starts with annotated stubs and a fake bridge, so the service and tests exist before the real pieces do. The communication work needs no code at all.

## Integration points (the only times people must sync)

| Point | Between | What happens |
|---|---|---|
| Contracts frozen | A, B, C | Annotation, interface, catalog, and test-vector format agreed |
| Bridge swap | A → B | The fake `CobolBridge` is replaced by the real one |
| End-to-end green | A, B, C | C's integration tests pass through the real agent and real COBOL |
| Pre-gate dry run | All | Clean checkout runs `demo.ps1` before M6 and M11 |

## Rules of thumb for working independently

- If a contract needs to change, change it in `contracts/` first and tell the others before changing code.
- Each person's work must pass its own tests without the other two present.
- Test-vector expected values always come from running real COBOL, never from a Java reimplementation of the COBOL logic.
- If someone is blocked for more than a day, the contract is probably unclear. Fix the contract, not the schedule.
