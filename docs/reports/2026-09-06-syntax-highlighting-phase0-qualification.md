# Syntax Highlighting Phase 0 Qualification

- Date: 2026-09-06
- Branch: `feature/syntax-highlighting`
- Baseline: `e7af9e4c24ac2ba6d49c3d301016a5d2c3ebab2e`

## 1. Baseline and environment

The branch was created from a clean `master` at the exact planning commit. `origin` is `https://github.com/harrywenjie/Notepads.git`; `upstream` is `https://github.com/0x7c13/Notepads.git`.

The workstation is Windows 11 build 10.0.26200.9278, x64, with 24 logical processors, Visual Studio 2022 Community MSBuild 17.12.36.22309, Windows SDK 10.0.26100, and .NET SDK 9.0.119. The application targets UWP build 17763 through 22621; Production uses .NET Native.

Untouched Debug x64 restore/build and Release x64 build passed. The known `MSB3277` conflicts remained:

- `System.Security.Principal.Windows` 4.1.1 versus 5.0.0
- `System.Security.AccessControl` 4.1.1 versus 5.0.0

Both arise from UWP framework references versus `Microsoft.Win32.Registry` 5.0.0. No new baseline warning was observed.

Local Debug deployment initially required the SDK copies of `Microsoft.NET.CoreRuntime.2.2` and `Microsoft.NET.CoreFramework.Debug.2.2`. After registration, untouched Debug and Release packages both exited during startup before an editor appeared, with Windows exception code `0xe0434352` and no managed stack in the Application log. The installed Store Notepads package was not removed or altered. This is a baseline local-deployment failure, not a spike regression. The temporary qualification entry point bypassed the failing multi-instance/startup route but loaded the real `App` resources and real editable `TextEditor`/`TextEditorCore` controls.

## 2. Experimental methodology

A temporary, compile-time-gated UWP runner was placed in `src/Notepads` and packaged under the isolated identity `Notepads-Phase0`. It constructed the repository's real `TextEditor`, installed it in `Window.Current`, called its normal `Init`, file, clipboard, edit, save, and session APIs, and accessed the actual `TextEditorCore : RichEditBox`. It did not use WPF, `RichTextBlock`, or a sample control.

Formatting always used independent ranges:

```csharp
var range = Document.GetRange(start, end);
range.CharacterFormat.ForegroundColor = color;
```

Direct calls and calls enclosed by balanced `BatchDisplayUpdates`/`ApplyDisplayUpdates` were measured. The runner never used `Document.Selection` as a paint range, never called `SetText` to color, never changed `UndoLimit`, never cleared undo except through normal fresh-document initialization, and never used undo groups to conceal formatting.

Evidence was written incrementally to package LocalState. All timings use `Stopwatch`. Memory readings use `Windows.System.MemoryManager.AppMemoryUsage`; they are process working-set-like snapshots affected by GC and are not allocation measurements. Generated inputs were deterministic. Temporary source, references, package identity changes, diagnostic UI, and generated samples were removed after recording this report.

## 3. Temporary instrumentation design

The runner counted `TextChanging` by `IsContentChanging`, `TextChanged`, selection events, and text-composition events. Each checkpoint captured selection endpoints, signed selection length/options, caret endpoint, horizontal/vertical scroll offsets, `CanUndo`, `CanRedo`, document text, `TextEditor` cached text, `IsModified`, `LastSavedSnapshot`, insertion foreground, and app memory.

The ColorCode spike directly constructed `LanguageRepository`, `LanguageCompiler`, and `LanguageParser`, then called `ILanguageParser.Parse`. The same call path ran in Debug and in an actually launched Production x64 .NET Native package. Parser and RichEditBox timings were kept separate.

## 4. RichEditBox event findings

Foreground formatting is reported as a format change, but it is not event-free. Coloring three independent ranges produced exactly three `TextChanging(IsContentChanging=false)` and three `TextChanged` events, with zero content-changing or selection events. Display batching produced the same counts. Resetting one whole-document range produced one of each format event; coloring 20,000 ranges produced 20,000 `TextChanging` events and normally 20,000 `TextChanged` events.

`TextEditorCore.OnTextChanging` correctly left its cached text untouched because `IsContentChanging` was false. `TextEditor` stayed unmodified and `LastSavedSnapshot` stayed byte-for-byte equivalent. However, every `TextChanged` enters the existing unconditional `OnTextChanged` path, including line-number rendering. A future range renderer would require event suppression/coalescing, but such a guard would not solve the undo failure and was not added.

## 5. Undo/redo findings

**Hard gate failure.** A fresh `abc` document was initialized through the normal editor path, one `X` was typed, and foreground formatting was applied. `CanUndo` was true after the edit and remained true after coloring. The first normal Undo changed only the foreground; text remained `abcX`. Redo restored the foreground, not text. The required expectation—one Undo removes `X`—failed both directly and with display batching.

Equivalent paste (`abcP`), replacement (`aZc`), deletion (`ac`), moved-caret, recolor, and clear-to-base tests all failed for the same reason: the first undo/redo operated on formatting rather than the user's text edit. Additional native-history evidence:

- formatting a clean document changed `CanUndo` from false to true;
- formatting after Undo cleared `CanRedo` (`true -> false`);
- recoloring and clearing each added formatting history;
- typing `X`, formatting, then typing `Y` split the native typing group: one Undo left `X`, whereas one Undo after typing `XY` without formatting left an empty document.

Therefore range foreground formatting creates native undo records, terminates typing coalescing, and destroys redo. No supported non-destructive suppression API was found or tested. Clearing undo, changing `UndoLimit`, or hiding formatting in an undo group is explicitly unacceptable.

## 6. Selection, caret, and insertion format

A reverse selection (`Start=350`, `End=500`, signed `Length=-150`, `StartActive`) remained exactly unchanged after 4,000 independent batched ranges; no selection events fired. The pass took 1,323.8 ms. Caret and selection endpoints also stayed stable in all size benchmarks.

Insertion formatting did not remain neutral. After coloring `token` red:

- typing inside the token inherited red;
- typing exactly at its start inherited red;
- typing exactly at its end inherited red;
- deleting across the token/plain boundary and continuing to type inherited red.

This creates persistent wrong color until another pass and a visible transient even if a later pass corrects it. Mixed selections can return a TOM mixed/automatic sentinel rather than one usable color. A renderer cannot assume that independent ranges leave the active insertion format at the base foreground.

## 7. Viewport findings

The isolated 4,000-range reverse-selection case at the top stayed at `(0,0)`. In the size benchmark, the caret was near character 100 and the viewport was programmatically placed lower in the document before formatting. Measured vertical deltas were:

| Approximate size | Direct | Batched |
|---:|---:|---:|
| 10 KiB | -46 px | -46 px |
| 100 KiB | -814 px | -814 px |
| 500 KiB | -4,227 px | -4,227 px |
| 1 MiB | -8,699 px | 0 px |
| 3 MiB | 0 px | 0 px |

Thus independent ranges do not move the selection, but formatting/event/layout processing can measurably return a scrolled viewport toward the caret. Batching is not a general cure. The test did not claim visual flicker measurement because no reliable frame telemetry was available.

## 8. Plain-text and byte integrity

Four cases covered ASCII without final newline; CJK, emoji/surrogate pairs, and combining characters with final CRLF; final CR; and final LF. Before and after coloring/recoloring/clearing, SHA-256 comparisons showed equality for:

- `Document.GetText(TextGetOptions.None)`;
- `TextEditor` cached text;
- `LastSavedSnapshot`;
- saved file bytes, including encoding and line-ending conversion;
- clipboard plain text copied through `TextEditor.CopyTextToWindowsClipboard`;
- text restored through the normal editor session-state reset path.

`IsModified` remained false for formatting-only passes. Formatted and unformatted cut operations produced the same remaining text. RichEditBox represented internal line breaks as CR and appended its terminal paragraph CR; parsing the exact internal UTF-16 snapshot produced valid offsets. Context-menu invocation and a full app close/relaunch were not automated; the baseline startup crash prevented a meaningful full-app restore test. The exercised editor save/session APIs nevertheless showed exact text and byte preservation.

## 9. IME and RTL findings

The runner attached `TextCompositionStarted`, `TextCompositionChanged`, and `TextCompositionEnded`, but programmatic edits correctly produced zero composition events. This environment could not operate Microsoft Pinyin's candidate UI or make a genuine interactive composition, so assigning Chinese strings was not misrepresented as IME evidence. No reliable interactive RTL editing pass was completed.

Human validation remains required if any future renderer is proposed:

1. Enable Microsoft Pinyin, type and revise an uncommitted composition in a colored token, choose a candidate, then Undo/Redo once.
2. Repeat while a scheduled recolor would become due; verify recoloring is deferred until composition ends.
3. Mix Chinese/English, backspace within composition, paste text, and verify caret/candidates do not jump.
4. Repeat selection, typing, deletion, copy/paste, Undo/Redo, and scrolling with Arabic or Hebrew mixed with LTR text.

REB-4 remains `PENDING HUMAN VALIDATION` as instructed, although its already-automated viewport and insertion-format subchecks found defects.

## 10. RichEditBox rendering benchmarks

Ranges followed realistic dense JSON-like keys, strings, numbers, and constants. Counts were capped at 20,000 for larger files. Times are UI-thread formatting time and exclude parsing.

| UTF-16 size | Ranges | Mode | Base reset | Color ranges | Total | Format `TextChanging` | `TextChanged` |
|---:|---:|---|---:|---:|---:|---:|---:|
| 10,277 | 956 | direct | 3.9 ms | 323.2 ms | 327.1 ms | 956 | 956 |
| 10,277 | 956 | batched | 3.9 ms | 254.4 ms | 258.3 ms | 956 | 956 |
| 102,426 | 9,528 | direct | 30.8 ms | 3,569.1 ms | 3,599.9 ms | 9,528 | 9,528 |
| 102,426 | 9,528 | batched | 23.8 ms | 2,425.5 ms | 2,449.3 ms | 9,528 | 9,528 |
| 512,001 | 20,000 | direct | 110.1 ms | 9,735.2 ms | 9,845.4 ms | 20,000 | 20,000 |
| 512,001 | 20,000 | batched | 111.8 ms | 5,482.0 ms | 5,593.9 ms | 20,000 | 20,000 |
| 1,048,598 | 20,000 | direct | 221.9 ms | 12,519.6 ms | 12,741.5 ms | 20,000 | 8,328* |
| 1,048,598 | 20,000 | batched | 224.5 ms | 5,986.1 ms | 6,210.6 ms | 20,000 | 20,000 |
| 3,145,757 | 20,000 | direct | 657.5 ms | 23,976.8 ms | 24,634.3 ms | 20,000 | 20,000 |
| 3,145,757 | 20,000 | batched | 685.4 ms | 7,661.1 ms | 8,346.5 ms | 20,000 | 20,000 |

`*` The asynchronous `TextChanged` count was sampled after a 150 ms settle and had not caught up; the synchronous format-changing count was complete.

Batching improved throughput but did not eliminate per-range events, undo records, or small/medium viewport movement. Even 100 KiB required about 2.45 seconds batched at this density. The proposed 500 KiB cutoff is therefore not conservative for this rendering method; the method is unacceptable well below it. Observed app-memory deltas ranged from approximately -9.8 MiB to +6.3 MiB across passes and are not allocation evidence.

## 11. ColorCode packaged-runtime result

`ColorCode.Core` 2.0.15 executed through the proposed parser API in both packaged configurations. The Debug package initialized the parser in 2.57 ms and completed the small JSON round trip in 62.17 ms on first process use. The launched Production x64 .NET Native package initialized in 0.088 ms, completed in 1.000 ms, round-tripped every UTF-16 code unit, produced four scopes, wrote a completion record, and exited normally. This is runtime—not build-only—evidence. No reflection/AOT/runtime-directive failure occurred.

The package remained the repository-pinned 2.0.15 dependency already transitively present through `Notepads.Controls`; no package upgrade survived. One API detail discovered during compilation is that JSON is registered in `Languages.All` but 2.0.15 exposes no `Languages.Json` property; lookup by `LanguageId.Json` is required.

## 12. ColorCode correctness and offsets

Every tested callback fragment concatenated to the exact immutable input, including CR, emoji (two UTF-16 code units), CJK, and combining characters. Accumulating callback fragment lengths and adding each fragment-relative `Scope.Index` produced only in-bounds absolute UTF-16 ranges; substring verification addressed the intended token. Applying LF-normalized offsets to RichEditBox was deliberately not tested or proposed.

| Case | UTF-16 length | Scopes | Invalid ranges | Round trip | Notable evidence |
|---|---:|---:|---:|---|---|
| JSON | 42 | 4 | 0 | pass | escaped string, exponent number |
| Python | 63 | 5 | 0 | pass | CR multiline triple string, comment |
| PowerShell | 64 | 6 | 0 | pass | command, variable, multiline comment |
| XML | 54 | 15 | 0 | pass | delimiter/name/attribute/comment |
| HTML | 55 | 16 | 0 | pass | tag/attribute and script content |
| CSS | 58 | 6 | 0 | pass | multiline comment/properties |
| JavaScript | 58 | 3 | 0 | pass | multiline comment/template-containing input |
| TypeScript | 63 | 4 | 0 | pass | interface/type/string |
| C++ | 65 | 3 | 0 | pass | preprocessor-containing C-like source/comment |
| C# | 53 | 4 | 0 | pass | class/string/comment |
| malformed JSON | 23 | 1 | 0 | pass | returned partial lexical result |
| incomplete Python | 24 | 1 | 0 | pass | returned partial lexical result |

The built-in samples returned top-level scopes only; the absolute-offset algorithm recursively supported children, but this corpus did not produce a non-zero nesting depth. ColorCode remains a lightweight lexical highlighter, not a correctness parser. Incomplete inputs did not throw.

## 13. ColorCode benchmarks

These results exclude RichEditBox rendering. “Cold” means the first parse at that size on a newly created parser for the language series; the first row includes first grammar compilation. Later sizes reuse its compiled grammar. Scope count indicates token density.

| Grammar | Size | First/size-cold | Warm | Warm scopes |
|---|---:|---:|---:|---:|
| JSON | 10 KiB | 1.87 ms | 1.62 ms | 1,398 |
| JSON | 100 KiB | 16.04 ms | 11.20 ms | 13,968 |
| JSON | 500 KiB | 55.16 ms | 54.26 ms | 69,822 |
| JSON | 1 MiB | 112.74 ms | 113.61 ms | 142,992 |
| JSON | 3 MiB | 336.97 ms | 336.97 ms | 428,964 |
| Python | 10 KiB | 2.21 ms | 0.02 ms | 2 |
| Python | 100 KiB | 0.12 ms | 0.13 ms | 2 |
| Python | 500 KiB | 0.61 ms | 0.60 ms | 2 |
| Python | 1 MiB | 1.10 ms | 1.06 ms | 2 |
| Python | 3 MiB | 3.27 ms | 4.25 ms | 2 |
| PowerShell | 10 KiB | 1.86 ms | 0.27 ms | 5 |
| PowerShell | 100 KiB | 2.72 ms | 2.48 ms | 5 |
| PowerShell | 500 KiB | 12.31 ms | 12.41 ms | 5 |
| PowerShell | 1 MiB | 26.23 ms | 25.86 ms | 5 |
| PowerShell | 3 MiB | 77.51 ms | 78.26 ms | 5 |
| JavaScript | 10 KiB | 0.76 ms | 0.28 ms | 2 |
| JavaScript | 100 KiB | 2.53 ms | 2.50 ms | 2 |
| JavaScript | 500 KiB | 12.73 ms | 12.36 ms | 2 |
| JavaScript | 1 MiB | 25.63 ms | 25.98 ms | 2 |
| JavaScript | 3 MiB | 77.05 ms | 78.90 ms | 2 |

The repeated Python/PowerShell/JavaScript generators created long repeated source patterns that the grammars captured in few large fragments; their scope densities are therefore not comparable to dense JSON. They measure scanning but should not be generalized as real-world token counts.

Bounded adversarial results:

| Input | Length | Result |
|---|---:|---|
| heavily escaped unterminated JSON string | 100,006 | exceeded 10 s watchdog |
| unterminated Python string | 100,009 | 47.9 ms |
| repeated JavaScript punctuation/comment-like pattern | 110,000 | exceeded 10 s watchdog |
| unterminated PowerShell block comment | 100,002 | 4.1 ms |

The timed-out `Task.Run` calls could not be cancelled. The watchdog moved to later cases, and no completion was observed before the packaged process exited at suite completion; the test deliberately did not wait indefinitely. In a separate dense 3 MiB JSON stale-work measurement, the result was marked stale at 30.8 ms but the synchronous worker continued until 630.6 ms, approximately 599.9 ms after supersession. This directly confirms that cancellation can only discard results, not stop CPU work. App-memory deltas were noisy and sometimes negative; no allocation claim is made.

## 14. Missing-language feasibility

Runtime enumeration confirmed no built-in YAML, Bash/shell, TOML, INI, JSONC, or distinct C grammar. Strict JSON did not color a `//` JSONC comment. The C++ grammar produced seven scopes on a representative C file, including basic keywords, but this does not prove full C/preprocessor correctness.

| Gap | Phase 0 classification |
|---|---|
| JSONC | Easy isolated strict-JSON-derived grammar with explicit line/block comments; strict JSON is not an acceptable approximation. |
| INI | Suitable for a very small custom tokenizer or isolated grammar; defer until corpus tests define section/key/comment rules. |
| TOML | Plausible isolated grammar, but multiline strings/dates require tests; not yet qualified. |
| YAML | Indentation, block scalars, anchors, and comments make a quick regex approximation risky; defer or qualify a dedicated small tokenizer. |
| Bash/shell | Quoting, expansions, heredocs, and comments make a small regex grammar high-risk; defer pending separate qualification. |
| C | C++ is an acceptable basic visual approximation only for a deliberately documented subset; do not claim C correctness or equivalence. |

No missing grammar was implemented, so none should be advertised as qualified v1 support.

## 15. RichEditBox decision matrix

| Gate | Result | Evidence |
|---|---|---|
| REB-1 — one-edit undo/redo integrity | **FAIL** | First Undo removed color, not the typed/pasted/replaced/deleted text. |
| REB-2 — native undo history unpolluted | **FAIL** | Formatting created undo records, split typing, and cleared redo; batching did not help. |
| REB-3 — event/dirty behavior manageable | **PASS** | Cached/saved/dirty state stayed intact, but every range emitted format-changing and changed events, so the implementation cost would be high. |
| REB-4 — IME/caret/selection/viewport | **PENDING HUMAN VALIDATION** | Genuine IME/RTL was unavailable; selection stayed stable, while insertion color inherited and viewport jumps were measured. |

**RichEditBox range-rendering strategy: `NO-GO`.** REB-1 and REB-2 are hard failures. The insertion, event-volume, rendering-time, and viewport findings reinforce but are not needed for that decision.

## 16. ColorCode decision matrix

| Gate | Result | Evidence |
|---|---|---|
| CC-1 — packaged runtime execution | **PASS** | Actual Debug and Production x64 .NET Native packages executed `ILanguageParser.Parse` successfully. |
| CC-2 — parser performance | **CONDITIONAL** | Normal bounded inputs were fast enough below 500 KiB, but two 100–110 KiB adversarial inputs exceeded 10 seconds. |
| CC-3 — language/correctness | **PASS FOR NARROWED BUILT-INS** | Exact UTF-16 offsets for ten built-ins; desired YAML/shell/TOML/INI/JSONC/C grammars remain absent. |
| CC-4 — startup/plain path and stale work | **CONDITIONAL** | Parser creation can be lazy, but active parses are synchronous/non-cancellable; dense stale work continued ~600 ms and pathological work >10 s. |

**ColorCode: `CONDITIONAL`.** It remains a technically compatible tokenizer candidate for a narrowed built-in set, not an unconditional preferred dependency. Any future use needs strict size/density policy, serialized workers, stale-result rejection, and protection against pathological regex input. A renderer failure does not invalidate the parser evidence.

## 17. Explicit architecture decision

Do not implement the plan's `RichEditBoxSyntaxRenderer` by assigning `ITextCharacterFormat.ForegroundColor` on the live editable document. There is no acceptable route from this spike to Phase 1/2 integration because native history is the application's only undo stack and formatting demonstrably corrupts its semantics.

ColorCode may remain behind a provider-neutral tokenizer interface if a safe presentation surface is later proven. That parser decision must remain independent of preview `FileType`, Markdown `Extensions`, and any eventual rendering technology. This phase does not recommend replacing the editor control; it records that the proposed in-place mechanism failed and requires a design review before feature work resumes.

## 18. Implications for Phase 1

Phase 1 must not begin under the current plan's assumption that Phase 0 passed. Before core abstractions become production code, revise the engineering plan around one of these evidence-backed outcomes:

1. identify and runtime-prove a supported non-undoable presentation API that does not mutate `ITextDocument` formatting; or
2. explicitly accept a separately designed editor/rendering approach only after assessing upstream divergence and all existing Notepads behavior.

Do not attempt to fix the gate by clearing native history, reducing `UndoLimit`, selection-based repainting, or undo grouping. If parser-only abstractions are explored later, restrict the initially qualified ColorCode set to JSON, Python, PowerShell, XML, HTML, CSS, JavaScript, TypeScript, C++, and C#. C can only be opt-in as an explicitly approximate C++ mapping after a broader corpus. JSONC, YAML, Bash, TOML, and INI remain unqualified.

## 19. Human validation and unresolved questions

- Genuine IME composition/candidate behavior and RTL editing remain untested by a human.
- Default rich clipboard/RTF payload was not decoded and compared; plain clipboard text was exact.
- The baseline local package startup exception needs separate diagnosis if ordinary local interactive deployment is required; it did not prevent isolated real-editor evidence.
- ColorCode's >10-second adversarial JSON/JavaScript behavior needs grammar-level diagnosis before it can be considered safe on arbitrary files.
- The tested corpus did not produce nested `Scope.Children`; nested precedence remains an adapter test requirement if ColorCode is revisited.
- No reliable per-pass allocation or flicker telemetry was available.

None of these open questions can reverse REB-1/REB-2; the current renderer remains no-go.

## 20. Exact reproduction commands

Run from `E:\Notepads` in Git Bash. `MSYS2_ARG_CONV_EXCL='*'` prevents MSYS from rewriting MSBuild switches.

```bash
MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' \
  src/Notepads.sln /t:Restore /p:Platform=x64 /p:Configuration=Debug /m /nologo /v:minimal

MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' \
  src/Notepads.sln /p:Platform=x64 /p:Configuration=Debug \
  /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /m /nologo /v:minimal

MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' \
  src/Notepads.sln /p:Platform=x64 /p:Configuration=Production \
  /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /m /nologo /v:minimal
```

For the disposable runtime run, the temporary harness described in sections 2–3 was compiled with `PHASE0_SPIKE`, packaged after building project references, and installed under a manifest publisher ending in the [Windows unsigned-package OID](https://learn.microsoft.com/en-us/windows/msix/package/unsigned-package):

```bash
MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' \
  src/Notepads/Notepads.csproj /p:Platform=x64 /p:Configuration=Production \
  /p:GenerateAppxPackageOnBuild=true /p:BuildProjectReferences=false \
  /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /nologo /v:minimal

powershell.exe -NoProfile -Command \
  'Add-AppxPackage -Path "E:\Notepads\src\Notepads\AppPackages\Notepads_1.5.6.0_x64_Production_Test\Notepads_1.5.6.0_x64_Production.msix" -AllowUnsigned'
```

The temporary harness itself is intentionally not retained; reproducing semantic measurements requires reintroducing equivalent compile-time-gated instrumentation against the exact methods listed above. The final committed tree contains only this evidence report, so ordinary product builds use the first three commands and behave as the planning baseline.
