# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

This is a **Java 8 / Maven 3.9+** Swing desktop app. The build is pinned strictly to JDK 8 via `maven-enforcer-plugin` and `maven-toolchains-plugin` — `~/.m2/toolchains.xml` must declare a `1.8` JDK or the build fails.

- `mvn clean package` — full build; produces shaded `target/Zettelkasten.jar`, plus `Zettelkasten.exe` (launch4j) and a macOS `.app` bundle when on the `mac` profile.
- `mvn test` — runs both test executions (see test layout below).
- `mvn -DskipTests package` — fast iteration when tests are unchanged.
- `mvn -Plocal-repo clean package` — build using the vendored `local-repository/` artifacts when external Maven mirrors are unavailable.
- `java -jar target/Zettelkasten.jar` — run the packaged app.
- `nix develop` — provisions JDK 8 + Maven + IntelliJ CE + writes `~/.m2/toolchains.xml` automatically (see `flake.nix`). Linux CI uses this; macOS CI uses Zulu 8 directly.

### Running a single test

The Surefire config has **three executions** (`default-test` is skipped; only `junit-tests` and `testng-tests` run). Both run during `mvn test`. To run a single test:

- JUnit 4 / Vintage: `mvn -Dtest=ClassName#methodName test` (only the `junit-tests` execution will pick it up).
- TestNG suites are driven by `src/test/resources/testng.xml`. Edit that suite or use `-Dsurefire.suiteXmlFiles=...` to scope.
- Tests run headless: `-Djava.awt.headless=true` is set globally by Surefire.

## Test Layout

Two parallel test stacks coexist and are intentionally isolated:

- **JUnit 4 (via JUnit 5 Vintage engine)** — most legacy tests; `junit-tests` execution forces `junit.platform.discovery.includeEngines=vintage` so the Jupiter engine doesn't try to discover Vintage tests itself.
- **TestNG 7** — separate `testng-tests` execution driven by `src/test/resources/testng.xml`.
- **JExample** — runs *under JUnit*, not TestNG. Keeping JExample out of TestNG's JUnit-mode discovery is a deliberate hygiene goal (see PR-SCOPE AC-07 in `AGENTS.md`).
- **Mockito (inline) + PowerMock** for isolation; PowerMock is JUnit-4-bound.

Test fixtures live in `src/test/resources/` (sample `.zkn3` files, settings files, `markdown-fixtures/conformance` and `markdown-fixtures/compatibility`).

## High-Level Architecture

Main entry point: `de.danielluedecke.zettelkasten.ZettelkastenApp` (registered as `exec.mainClass`). Despite its name, `ZettelkastenAppRefactor.java` is a parallel refactor target — both exist; do not assume one supersedes the other without checking.

### Package layout (under `de.danielluedecke.zettelkasten`)

The root package is dense with **Swing dialog classes** (each `C*.form` + `C*.java` pair is a NetBeans Matisse GUI form + its generated/edited code: `CExport`, `CImportBibTex`, `CSettingsDlg`, `EditorFrame`, `DesktopFrame`, `ZettelkastenView`, etc.). The `.form` XML files are authoritative for layout — do not hand-edit generated initComponents() bodies without re-syncing.

Subpackages:

- `database/` — core domain model. `Daten` is the central data class; it stores everything in a **JDOM2 XML tree** (the `.zkn3` file format). `Daten.currentVersion` and `backwardCompatibleVersion` gate file-format compatibility. `BibTeX`, `DesktopData`, `Bookmarks`, `Synonyms`, `SearchRequests`, `TasksData`, `StenoData`, `AutoKorrektur`, `Document` live here. The `*UiCallbacks` interfaces (`DatenUiCallbacks`, `BibTeXUiCallbacks`) are the **decoupling seam** that lets non-UI code call back into Swing without depending on it directly — see PR-SCOPE AC-06.
- `data/` — `History`, `SearchResultsFrameData` (lighter-weight data holders).
- `tasks/` — long-running operations as `SwingWorker`-style background tasks (`SaveFileTask`, `AutoBackupTask`, `RefreshBibTexTask`, etc.). Subpackages: `tasks/export/` (Markdown, HTML, LaTeX, CSV, TXT, XML, ZKN exporters), `tasks/importtasks/`, `tasks/search/`. Each task typically has a `TaskProgressDialog`-driven UI.
- `util/` — utilities. Notable: `HtmlUbbUtil` (UBB ↔ HTML conversion — UBB is the internal markup), `MarkdownWorkspaceExporter` (Pandoc-driven workspace export), `HtmlValidator`, `UbbNestingNormalizer`/`UbbNestingValidator`, `Constants` (holds the shared `zknlogger`).
- `view/`, `ui/`, `tags/`, `walks/`, `history/`, `settings/`, `config/`, `mac/` — UI/view layer, settings persistence, tag handling, macOS integration shims.

### Key cross-cutting concerns

- **Format pipeline**: notes are authored in UBB (BBCode-like), stored as XML, projected to HTML for the Swing JEditorPane, and exported to Markdown/LaTeX/etc. Conversion utilities in `util/HtmlUbbUtil`, `util/UbbNestingNormalizer`, and the markdown lint code in `src/test/java/.../markdownlint/` (`MarkdownCompatibilityLinter`, `MarkdownSupportMatrix`) are the load-bearing pieces. See `docs/markdown-support.md` for the lint rule matrix.
- **Workspace Markdown export**: on each Zettel save, `MarkdownWorkspaceExporter.exportOnSave` writes `z<entryNumber>.md` to the workspace dir. Resolution order: `ZETTELKASTEN_WORKSPACE_DIR` env var → `${user.home}/workspace` if it exists → no-op (logs INFO once). Pandoc invocation: `pandoc -f html -t markdown -o <out> <tmpHtml>`. Failures must not interrupt save — log `WARNING` and continue. See PR-SCOPE AC-08.
- **Logging**: use `Constants.zknlogger` (`java.util.logging`). `logback-classic` is on the classpath but the project standardizes on JUL; do not introduce a new logging facade without coordinating in `pom.xml`.
- **Headlessness**: tests assume `java.awt.headless=true`. Core data/model classes should be instantiable headlessly — Swing references in `database/` are being actively removed (PR-SCOPE AC-06).

### Build/packaging plugins

- `maven-shade-plugin` — produces the runnable fat JAR at `target/Zettelkasten.jar`.
- `launch4j-maven-plugin` — wraps the JAR as `Zettelkasten.exe` on every build.
- `macosappbundler-maven-plugin` — builds the macOS `.app`; DMG generation is gated by `-Dmacosappbundler.dmg.enabled=true`.
- `git-commit-id-plugin` — embeds Git metadata into the build (runs in `offline` mode).

## Repository Conventions (from AGENTS.md)

This repo uses **explicit PR scopes** (AC-01 through AC-09 in `AGENTS.md`) — each PR must declare and stay within one scope. Codex/agent work follows "one prompt = one PR". Forbidden across most scopes: UI redesign, persistence-format changes (`.zkn3` schema), feature additions beyond the scope. Read `AGENTS.md` before non-trivial changes.

- **Conventional Commits**: `feat(scope): ...`, `fix(scope): ...`, `docs(scope): ...`. Subject under 72 chars.
- **Code style**: 4-space indent, K&R braces, one public class per file. Prefer `final` for immutable fields. Keep package names under `de.danielluedecke.zettelkasten` or `ch.dreyeck.zettelkasten`.
- **Resources**: UI strings, icons, and properties belong in `src/main/resources`. Avoid hard-coded absolute paths. Properties files get Maven filtering; everything else does not.
- **Test data**: keep deterministic, no network or filesystem writes outside `target/`. Sample zettels (e.g. `validFile.zkn`) belong in `target/` or LFS, not under version control.
