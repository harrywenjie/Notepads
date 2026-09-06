# Syntax Highlighting Engineering Plan

Date: 2026-09-06

Status: design; production implementation is not authorized

Proposed implementation branch: `feature/syntax-highlighting`

## 1. Executive summary

Use the existing `RichEditBox` only if a Phase 0 runtime spike proves that range foreground formatting is invisible to normal undo/redo, content-change/dirty tracking, caret/selection, IME composition, and scrolling. Microsoft documents the formatting and text APIs, but does **not** document those interactions. They are a hard go/no-go gate, not details to discover after integration.

If that gate passes, retain `RichEditBox` and introduce replaceable layers for language detection, parsing, neutral semantic spans, palette selection, scheduling, and UI-thread range rendering. The provisional parser recommendation is a `ColorCode.Core` adapter for only the languages it supports well, supplemented or replaced over time without changing editor integration. ColorCode is already compatible at the package/framework level and is small, but it is a synchronous whole-document regex parser, has no cancellation or incremental API, has had no release since 2.0.15 in July 2023, and omits YAML, shell, TOML, and INI. It is therefore **conditionally qualified, not pre-approved**.

The initial product must stay automatic and opt-out: no manual language selector, IDE services, project model, diagnostics, execution, or editor-control replacement. Unknown and `.txt` files do no syntax work. Large recognized files use a conservative size gate rather than re-highlighting blindly.

## 2. Current repository facts

### Baseline and build

- The inspected baseline is `master` at `801b74c57ad0d30b9760bc207ef4437c6f539d48`; the working tree was clean before this document was created. `origin` fetch/push is `https://github.com/harrywenjie/Notepads.git`. `upstream` exists and fetch/push is `https://github.com/0x7c13/Notepads.git`; it has not been changed or fetched during planning.
- `src/Notepads.sln` contains two legacy, non-SDK UWP projects: `Notepads` (the packaged app) and `Notepads.Controls` (the vendored Markdown/control library). The app targets UAP `10.0.22621.0`, minimum `10.0.17763.0`, and x86/x64/ARM64 in Debug, Release, and Production. Packaging is MSIX; `Package.targets` changes non-Production identity to `Notepads-Dev`.
- On this Windows 11 workstation, Visual Studio 2022 Community 17.12, the UWP workload, and Windows SDKs 22621/26100 are installed. The repeatable baseline commands from Git Bash are:

  ```bash
  MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' src/Notepads.sln /t:Restore /p:Platform=x64 /p:Configuration=Debug /m /nologo /v:minimal
  MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' src/Notepads.sln /p:Platform=x64 /p:Configuration=Debug /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /m /nologo /v:minimal
  ```

- Restore was up to date and the Debug x64 build succeeded. A planning-only Production x64 build with the same signing/bundle overrides also completed .NET Native code generation and emitted an MSIX. Both configurations showed the same existing `MSB3277` conflicts: `System.Security.Principal.Windows` 4.1.1 versus 5.0.0 and `System.Security.AccessControl` 4.1.1 versus 5.0.0, arising from UWP reference assemblies and `Microsoft.Win32.Registry` 5.0.0. These are baseline warnings, not feature regressions.
- There is no normal automated test project. Existing validation is solution builds plus manual app exercise. Ignored build/package outputs did not dirty the working tree.

### Architecture and dependency boundaries

- `src/Notepads` contains app shell, tabs/session management, file I/O, settings, and the editable `RichEditBox`. `src/Notepads.Controls` contains a forked Markdown renderer and must remain reusable; editor highlighting does not belong there.
- Direct app dependencies include DiffPlex, UWP platform packages, Toolkit UI, Win32 Registry, XAML Behaviors, Newtonsoft.Json, System.Text.Json, and UTF.Unknown. `Notepads.Controls` already references `ColorCode.UWP` 2.0.15, Toolkit UI, and Win2D. Do not upgrade or move these packages merely for this feature.
- `FileSystemUtility.ReadFileAsync` detects encoding/BOM and line endings and applies an approximately 1,000 KiB normal-open limit unless explicitly bypassed. `WriteText` reapplies the chosen encoding preamble and line-ending conversion. Highlighting must operate on the in-memory editor snapshot and never enter this byte/encoding pipeline.
- `FileType` currently means only `Unknown`, `TextFile`, or `MarkdownFile` and controls preview eligibility. `INotepadsExtensionProvider` returns a Markdown `IContentPreviewExtension` that binds a second read-only preview to `TextEditorCore`; it is a content-preview seam, not an editor plugin host. Neither abstraction should carry syntax language state.

### Editor and document lifecycle trace

- `NotepadsCore.CreateTextEditorAsync` reads a `TextFile`, constructs `TextEditor`, injects the preview extension provider, calls `Init`, and subscribes to the `ITextEditor` lifecycle. Tab deletion unsubscribes and calls `Dispose`. `ITextEditor` deliberately exposes editor-level text/file/lifecycle operations and events, but not `RichEditBox.Document`; keep that boundary.
- `TextEditor` (`Controls/TextEditor/TextEditor.xaml.cs`) owns `StorageFile`, display name/path, preview `FileType`, requested encoding/line ending, `LastSavedSnapshot`, dirty state, polling for external changes, preview/diff state, and the `TextEditorCore`. `Init` sets text, snapshot and undo state while `_loaded` suppresses dirty propagation. `Reload` rereads and reinitializes. `ResetEditorState` restores session text, selection, wrapping/font, encoding/line ending and scroll state. `RenameAsync`, Save As (save to a new `StorageFile` followed by `Init`), and ordinary save all flow through `UpdateDocumentInfo`; that is the existing filename-change seam for redetection.
- `TextEditorCore : RichEditBox` (`TextEditorCore.cs`/`.xaml`) owns the native `ITextDocument`, selection, keyboard commands, find/replace mutations, line numbers/highlight, scroll state and themed base foreground. Its internal text uses `\r`; `OnTextChanging` retrieves `Document.GetText(TextGetOptions.None)`, trims the terminal paragraph `\r`, updates its cache only when `IsContentChanging`, and invalidates line data. `OnTextChanged` redraws line numbers. Its `Undo`/`Redo` directly call native `Document.Undo`/`Redo`; there is no application-owned edit stack.
- Small mutations use `Selection.SetText`/`TypeText`; operations such as whole-document transforms use `Document.SetText`. Plain paste is wrapped in `BeginUndoGroup`/`EndUndoGroup`. A historical IME/RTL fix set `Selection.CharacterFormat.TextScript = TextScript.Ansi`, but that formatting line was later commented out to fix another input issue. This is direct repository evidence that character formatting can interact with input and must be gated.
- `TextEditor.TextEditorCore_OnTextChanging` ignores non-content changes and events while not loaded, then compares the cached text with `LastSavedSnapshot` to set `IsModified`. Save reads the cached plain text, converts only through existing line-ending/encoding logic, writes, and refreshes the snapshot without resetting text. The highlighter should schedule after content `TextChanged`, not perform work inside synchronous `TextChanging`.
- Theme resources in `TextEditorCore.xaml` provide light, dark, and high-contrast base foreground. `ThemeSettingsService.OnThemeChanged` is already subscribed by `TextEditor` (currently to rerender diff), while `ExternalEventListener` updates selection colors on accent changes. A highlighter session can use the same theme notification but must detach it in `Dispose`.
- Markdown preview is lazy, secondary rendering driven by `IContentPreviewExtension`; diff mode disables the editor and renders `LastSavedSnapshot` versus current plain text in `SideBySideDiffViewer`. Highlighting attaches only to the editable `TextEditorCore`. It must not color preview/diff controls or change preview `FileType`, and pending application should be suspended while the editor is unloaded/disabled.
- `SessionManager` serializes plain text plus editor metadata and restores through `ResetEditorState`; file identity may be restored before or after text state. Syntax state is derived and should never be serialized. Restoration must invalidate/redetect/reparse once the final filename and text generation are installed.

### Upstream constraint

- Upstream issue [#74](https://github.com/0x7c13/Notepads/issues/74) is still open. The owner explicitly declined the feature at the time because syntax coloring consumes CPU, may increase launch time, and complicates large-file handling ([owner comment](https://github.com/0x7c13/Notepads/issues/74#issuecomment-513914061)). Those concerns are acceptance constraints for this fork, not background commentary.
- The request is chiefly useful for ad-hoc configuration and script editing (JSON, YAML, XML, and similar), consistent with later discussion on that issue. It must not turn Notepads into an IDE.

## 3. Requirements and non-goals

The feature is presentation-only. It must preserve saved bytes (apart from an explicit user edit or existing encoding/line-ending behavior), text, whitespace, selection, clipboard plain text, sessions, dirty state, and user undo history. It must be lazy for unrecognized/plain text, safe under stale asynchronous work, theme-aware, optional globally, and bounded for large files.

Non-goals are autocomplete, IntelliSense, parsing diagnostics, linting, code execution, folding, project/workspace behavior, a general extension host, and replacing the editor engine unless `RichEditBox` fails the hard integrity gate.

## 4. Research findings

### Upstream request

The issue author asked for the speed of a simple Notepad replacement plus syntax coloring. The repository owner responded that Notepads should remain Notepad-like and named CPU usage, launch time, and large-file behavior as reasons not to add highlighting ([issue and owner response](https://github.com/0x7c13/Notepads/issues/74#issuecomment-513914061)). The design must demonstrate that plain-text launch/edit paths are essentially unaffected and must fail closed on large files.

### Evidence rules

Microsoft API documentation establishes what public APIs mean. It does not establish undocumented interactions between formatting, undo, XAML events, IME, viewport, or selection. ColorCode's repository and package metadata establish available APIs and assets, not production performance in Notepads. All such gaps below remain Phase 0 experiments.

### Existing and historical ColorCode evidence in Notepads

`src/Notepads.Controls/Notepads.Controls.csproj` already pins `ColorCode.UWP` 2.0.15. Its only current repository use is the vendored Markdown preview path: `MarkdownTextBlock` enables syntax coloring for fenced code, `ITextBlockResolver.ParseSyntax` finds a ColorCode language, and `RichTextBlockFormatter.FormatInlines` creates spans in a read-only `RichTextBlock`. That proves a preview renderer exists; it neither edits nor formats `TextEditorCore.Document`.

Git history attributes this code to the 2020 import of Windows Community Toolkit Markdown controls (commit `8e3cc3a`), not to a prior Notepads editor-highlighting feature. Searches across current source/history found no ColorCode-based editable-document renderer. Older generic `IContentExtension`/diff-provider shapes introduced in 2019 were later removed as unused, leaving the current Markdown-only preview contract. Therefore there is reusable parser/package evidence, but no historical solution to RichEditBox undo, input, or range rendering.

## 5. RichEditBox technical findings

### What the public contract establishes

- `RichEditBox.Document` exposes the Windows Text Object Model. `ITextDocument.GetRange(start, end)` returns a new range, while `ITextDocument.Selection` is the active selection ([`ITextDocument`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument), [`GetRange`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.getrange), [`Selection`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.selection)). Rendering must use independent `GetRange` objects, never temporarily repurpose the active selection. Changing positions on a range that *is* the active selection can scroll it into view ([`ITextRange.StartPosition`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange.startposition)).
- `ITextRange.CharacterFormat` is the formatting object for a range, and `ITextCharacterFormat.ForegroundColor` gets/sets its text color ([`CharacterFormat`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange.characterformat), [`ForegroundColor`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextcharacterformat.foregroundcolor)). These pages do not promise that the operation is non-undoable or event-free.
- `ITextRange.Text` is explicitly plain text. `TextGetOptions.FormatRtf` must be requested to retrieve RTF; `None` is plain text ([`ITextRange.Text`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange.text), [`TextGetOptions`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.textgetoptions)). This strongly supports text-content integrity, but byte-for-byte saves still require an application-level test because Notepads also normalizes its internal newlines and reapplies encoding/BOM/line-ending policy.
- Rich-text controls have a final end-of-paragraph marker. `TextGetOptions.UseCrlf` replaces a carriage return with CRLF, `UseLf` replaces carriage returns with LF, and `AdjustCrlf` recognizes CRLF, surrogate pairs, variation selectors, and table delimiters as constructs ([`TextGetOptions`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.textgetoptions)). The integration must parse the same UTF-16 string whose offsets it renders, excluding the editor's final marker. Do not parse a file-normalized LF string and apply those offsets to a CR-based RichEditBox document.
- `TextChanging` is synchronous, occurs before rendering, and sees the new document value. Microsoft advises limiting it to document inspection/update because visual-tree changes can crash in contexts such as layout. `TextChanged` is asynchronous and occurs after rendering ([`RichEditBox.TextChanging`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richeditbox.textchanging)). Crucially, Microsoft says `TextChanging` fires for format **or** content changes and `IsContentChanging` distinguishes them ([`IsContentChanging`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richeditboxtextchangingeventargs.iscontentchanging)). The baseline already guards its text cache and dirty-state propagation with that property ([`TextEditorCore.OnTextChanging`](https://github.com/harrywenjie/Notepads/blob/801b74c57ad0d30b9760bc207ef4437c6f539d48/src/Notepads/Controls/TextEditor/TextEditorCore.cs#L375-L382), [`TextEditorCore_OnTextChanging`](https://github.com/harrywenjie/Notepads/blob/801b74c57ad0d30b9760bc207ef4437c6f539d48/src/Notepads/Controls/TextEditor/TextEditor.xaml.cs#L920-L932)). This is positive evidence that formatting-only `TextChanging` events should not dirty the document. Parsing and high-volume coloring still do not belong inside that synchronous callback.
- `BatchDisplayUpdates` and the matching `ApplyDisplayUpdates` maintain a counter that suppresses intermediate display updates, reducing flicker and repeated render cost; every increment must be balanced in `finally` ([`BatchDisplayUpdates`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.batchdisplayupdates), [`ApplyDisplayUpdates`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.applydisplayupdates)). These APIs batch display only; the documentation gives no undo or event-suppression guarantee.
- `BeginUndoGroup` groups editing anti-events so one Ctrl+Z undoes the whole group and also suppresses screen updates ([`BeginUndoGroup`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.beginundogroup)). It is not an undo-suppression API. It could combine color operations with a user edit, so it must not be adopted as the solution. `UndoLimit` merely states the maximum queue length; UWP documentation does not guarantee that lowering and restoring it preserves existing history ([`UndoLimit`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument.undolimit)).
- Copying from a rich edit control normally makes both plain-text and RTF clipboard formats available. `ClipboardCopyFormat=PlainText` can force plain text only ([`RichEditClipboardFormat`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richeditclipboardformat), [`ClipboardCopyFormat`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richeditbox.clipboardcopyformat)). Foreground formatting should not alter the plain-text payload, but it will be observable in RTF under the default all-formats setting. The feature should at least assert plain-text equality; forcing plain-only copy is a separate product decision and may be unnecessary if Notepads already does so.
- `RichEditBox` is a `DependencyObject`; UWP XAML dependency objects have UI-thread affinity and non-dispatcher calls from worker threads throw ([`DependencyObject` threading](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.dependencyobject)). Snapshot immutable text on the UI thread, parse that string off-thread, and apply `Document`/range formatting only on the editor's UI thread.
- The native desktop TOM defines `tomApplyTmp` for temporary formatting ([`tomConstants`](https://learn.microsoft.com/en-us/windows/win32/api/tom/ne-tom-tomconstants)), but the UWP `ITextRange`/`ITextCharacterFormat` surface used by this project exposes no corresponding apply-mode operation. Desktop COM/TOM APIs must not be assumed usable from this packaged UWP control.

### What documentation does not resolve

These are all **unresolved hard evidence gaps**:

1. Does setting `ForegroundColor` append an undo record, terminate a typing undo group, clear redo, or change `CanUndo`?
2. Formatting is documented to raise `TextChanging` with `IsContentChanging=false`; does it also raise `TextChanged` or `SelectionChanged`, and what is the cost of the baseline's unconditional line-number rendering in `OnTextChanged`? Does the observed event sequence confirm the documented flag and current dirty-state guard?
3. Does formatting via an independent range preserve active selection direction, caret affinity, insertion formatting, scroll offsets, and viewport under hundreds/thousands of ranges?
4. Does typing within/at the boundary of a colored range inherit token color in a way that becomes visible before the next render?
5. Can a formatting pass safely coexist with active IME composition, candidate UI, handwriting, RTL text, and undo grouping?
6. Does a standard copy preserve plain text exactly for CR, CRLF, supplementary characters, combining marks, and selected final paragraph boundaries?
7. What is the UI-thread application cost and flicker behavior at realistic span counts and file sizes?

### Required disposable Phase 0 semantics spike

Build a minimal test page or temporary instrumented code path, then remove it before production work. Use the actual Notepads `TextEditorCore`, not a different framework's rich-text control. For each direct `range.CharacterFormat.ForegroundColor` strategy (with and without balanced display batching):

1. Load known text; capture `GetText(None)`, Notepads' cached text, dirty flag, selection start/end, scroll offset, `CanUndo`/`CanRedo`, and event counters.
2. Type exactly one character, let coloring run, press Ctrl+Z once, and prove only that character is removed. Redo must restore only that character. Repeat after paste, replace, and a selection move.
3. Apply/reapply/clear hundreds and thousands of ranges. Assert no content events that drive dirty state, no selection/caret/scroll jump, and no undo/redo change.
4. Compare `GetText(None)`, the plain clipboard payload, session text, and bytes saved with highlighting off/on/recolored. Also inspect whether RTF is intentionally present.
5. Exercise IME composition manually while the debounce expires. The safe default is to defer all application from `TextCompositionStarted` until `TextCompositionEnded`, then schedule one fresh generation.
6. Run light/dark/high-contrast recoloring and disable/enable cycles. Clearing must restore the current editor foreground, not an old theme color.
7. Run at 10 KB, 100 KB, 500 KB, 1 MB, and several MB while recording UI application time, event counts, selection/scroll deltas, and undo state.

**Gate REB-1:** no production renderer work may begin unless the one-character undo scenario passes using only supported UWP APIs.

**Gate REB-2:** if formatting is undoable or corrupts edit undo groups and no supported non-destructive suppression mechanism exists, reject in-place RichEditBox highlighting. Do not ship by clearing the undo buffer or toggling `UndoLimit`.

**Gate REB-3:** if formatting causes content events, a narrow renderer reentrancy guard may prevent internal scheduling/dirty recalculation only after proving actual text is unchanged; it cannot repair polluted native undo history.

**Gate REB-4:** if IME or viewport integrity cannot be made reliable by deferring/coalescing application, stop and revisit the rendering strategy.

## 6. ColorCode qualification

### Package and project state

- The authoritative repository is [CommunityToolkit/ColorCode-Universal](https://github.com/CommunityToolkit/ColorCode-Universal). It is not archived, is MIT licensed, and separates core parsing from UWP/WinUI/HTML formatters ([README and license](https://github.com/CommunityToolkit/ColorCode-Universal#readme)). The latest release and NuGet version are [2.0.15, published 2023-07-14](https://github.com/CommunityToolkit/ColorCode-Universal/releases/tag/v2.0.15). Main's latest code commit is also from July 2023; repository metadata shows a later push but no newer release. Treat it as stable/low-activity, not actively evolving.
- [`ColorCode.Core`](https://www.nuget.org/packages/ColorCode.Core/2.0.15) contains `netstandard1.4` and `netstandard2.0` assets. The `netstandard2.0` group has no runtime package dependencies. Inspection of the signed package gives a roughly 122 KB core DLL per TFM.
- [`ColorCode.UWP`](https://www.nuget.org/packages/ColorCode.UWP/2.0.15) targets `uap10.0.17763`, depends only on `ColorCode.Core >= 2.0.15`, and describes itself as a `RichTextBlock` renderer. Its DLL is roughly 21 KB. Notepads' restored assets currently select `ColorCode.Core`'s `netstandard2.0` assembly for UAP, including AOT target graphs.
- ColorCode's source contains no reflection, `dynamic`, `Activator`, or runtime assembly discovery in the core/UWP projects. Built-in language types are registered directly in [`Languages`](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Languages.cs). The UWP package contains an empty library `rd.xml`, which requests no special reflection preservation ([runtime directives](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.UWP/Properties/ColorCode.UWP.rd.xml)). This reduces AOT risk but is not proof of Release/Production .NET Native execution. Microsoft notes that .NET Native lacks a JIT and requires runtime directives for dynamic/reflection patterns it cannot infer ([.NET Native reflection](https://learn.microsoft.com/en-us/windows/uwp/dotnet-native/reflection-and-net-native)).

### Parser API and span usability

`ILanguageParser.Parse(string, ILanguage, Action<string, IList<Scope>>)` is synchronous ([interface](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Parsing/ILanguageParser.cs)). The implementation builds/caches one combined regex per language, scans the entire supplied string, and invokes the callback for unmatched and matched fragments ([parser](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Parsing/LanguageParser.cs), [compiler](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Compilation/LanguageCompiler.cs)). Each `Scope` has `Name`, `Index`, `Length`, nested children, and parent ([`Scope`](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Parsing/Scope.cs)). Scope indexes are relative to each callback fragment, so an adapter can accumulate fragment lengths into absolute UTF-16 positions and flatten nested scopes according to an explicit precedence rule.

Core does not require a renderer. Only `ColorCode.UWP.RichTextBlockFormatter` assumes `RichTextBlock`/`InlineCollection` ([source](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.UWP/RichTextBlockFormatter.cs)). Notepads should consume the Core parser through its own adapter and must not use that formatter for the editable control.

### Language coverage

Built-ins are ASP.NET variants, C#, C++, CSS, F#, HTML, Java, JavaScript, JSON, TypeScript, PHP, PowerShell, SQL, VB.NET, XML, Koka, Haskell, Markdown, Fortran, Python, and MATLAB ([IDs](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Common/LanguageId.cs)). Relevant requested gaps are YAML, Bash/shell, TOML, INI, JSONC, and a distinct C grammar. C might be acceptably approximated by C++ only after examples prove it; JSONC must not silently use strict JSON because comments would remain wrong.

### Performance, allocation, and lexical-state implications

- There is no incremental parse API, prior-state input/output, cancellation token, time budget, or asynchronous API. A stale parse result can be discarded, but an already-running regex parse cannot be cooperatively stopped.
- The parser constructs substrings for every unmatched and matched fragment and allocates scope lists/trees. Memory and GC cost therefore scale with document size and match density; the project publishes no applicable Notepads benchmark.
- The first parse of a language compiles its combined grammar regex and caches it behind reader/writer locks. Accessing the static `Languages` catalog constructs all built-in language definitions, but does not compile every language regex. Keep first access off app startup and measure cold first-open separately from warm parses.
- It parses the complete supplied string, so multiline strings/comments work only to the extent that each language regex models them. ColorCode exposes no resumable per-line lexical state. Parsing only changed lines would break cross-line constructs; whole-document parsing is the only correctness-preserving use of the current API.
- The compiled language regex itself is constructed without `RegexOptions.Compiled`; only ColorCode's helper regex for counting capture groups uses that option ([compiler source](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Compilation/LanguageCompiler.cs)). Do not assume JIT-compiled language regex throughput.
- The grammars are lightweight lexical regexes rather than standards-complete parsers. Malformed/incomplete input and complex edge cases require corpus tests. For example, JSON 2.0.15 recognizes keys, strings, numbers, and constants but not comments ([JSON grammar](https://github.com/CommunityToolkit/ColorCode-Universal/blob/main/ColorCode.Core/Compilation/Languages/Json.cs)).

### Qualification result

**Conditional candidate for approach A.** It satisfies license, binary-size, UWP TFM, renderer independence, and usable-span requirements. The unchanged repository's successful Production x64 .NET Native build proves its existing binaries survive current AOT compilation. It does not satisfy requested language coverage, incrementality/cancellation, packaged execution through the proposed parser path, or measured performance by evidence alone.

**Gate CC-1:** a disposable Phase 0 spike must reference/use `ColorCode.Core` through the same legacy UWP project, restore, and compile Debug plus Release/Production x64 with .NET Native. Production x64 compilation is already baseline-proven; the spike must additionally execute parser calls in a packaged Release/Production build because restore/build alone is insufficient.

**Gate CC-2:** measure cold initialization, warm parsing, allocations/memory, malformed input, and representative token counts for 10 KB through several MB. Include catastrophic/very slow regex protection at the scheduling policy level because parsing has no cancellation.

**Gate CC-3:** validate every initially advertised language against a small checked-in corpus. Do not claim unsupported grammars; either narrow v1 coverage or add isolated adapter-compatible lexers later.

**Gate CC-4:** if parsing cannot remain off the startup/plain-text path, or if a stale multi-megabyte parse consumes unacceptable CPU after supersession, do not select ColorCode.

## 7. Alternatives considered

### A. ColorCode.Core plus a Notepads range renderer

Advantages are small managed payload, an existing compatible UWP package family, MIT licensing, broad coverage of mainstream web/.NET languages, and a parser/rendering split. Disadvantages are whole-document synchronous regex work, allocations, missing requested config/shell grammars, low maintenance activity, no cancellation, and grammar-level correctness limits. This is the provisional recommendation only after `REB-*` and `CC-*` gates pass.

### B. Small Notepads-owned lexer architecture

A small stateful per-line lexer can give explicit cancellation, line-state propagation for multiline constructs, low dependencies, and cheap extension/filename mapping. It also transfers correctness and security maintenance to this fork across at least sixteen requested languages. It is reasonable only for a deliberately narrow first release (for example JSON/JSONC, INI, TOML, and perhaps YAML subsets) behind the same parser interface, not as an immediate promise to implement every language from scratch.

### C. Other libraries

[TextMateSharp 2.0.4](https://www.nuget.org/packages/TextMateSharp/2.0.4) is current, MIT, targets .NET Standard 2.0, exposes per-line tokenization with a returned rule stack (therefore proper cross-line lexical state), and its grammar package covers the target languages ([project README](https://github.com/danipen/TextMateSharp#readme)). It is not qualified for this legacy UWP app: it depends on `Onigwrap`, whose package carries P/Invoke native binaries for desktop-style Windows RIDs but declares no UAP target ([Onigwrap package](https://www.nuget.org/packages/Onigwrap/1.0.11)); the grammar payload is materially larger; and no authoritative source claims UWP AppContainer or .NET Native compatibility. It would need its own packaged x86/x64/ARM64 and Production AOT spike. Its footprint and native deployment risk make it a fallback research candidate, not the selected option.

Tree-sitter bindings are a still higher-cost native option: current packages bundle many native parsers and are tens of megabytes ([TreeSitter.DotNet](https://www.nuget.org/packages/TreeSitter.DotNet)). That conflicts with the lightweight/upstream-friendly goal and has no demonstrated UWP qualification.

No other third-party library was qualified from authoritative evidence. WPF/Avalonia editor libraries do not establish compatibility with this UWP `RichEditBox` application.

### D. Replace the editor engine

An editor-control replacement could provide temporary decorations and incremental syntax services, but it would rewrite the behavioral center of Notepads and risk selection, IME, scrolling, touch/pen, accessibility, undo, and upstream rebases. Consider it only if `REB-2` fails and syntax highlighting remains worth a separate long-term fork architecture; it is a no-go for the initial implementation plan.

## 8. Selected architecture and rationale

Subject to Phase 0 gates, keep `RichEditBox` and use a provider-neutral pipeline:

`filename + immutable text snapshot -> language detector -> ISyntaxTokenizer -> SyntaxSpan[] -> generation check -> theme palette -> RichEditBox range renderer`

This is one deep module with a small editor-facing interface: `SyntaxHighlightingSession` owns detection, scheduling, supersession and rendering for one `TextEditor`. Its internals contain replaceable seams, while `TextEditor` only supplies lifecycle signals and exact snapshots. The tokenizer must know nothing about XAML. The renderer must know nothing about ColorCode scopes. `ColorCodeSyntaxTokenizer` is an adapter that converts callback fragments/nested scopes into sorted, validated, non-overlapping neutral spans `(start, length, SyntaxTokenKind)`. A fake tokenizer used by tests makes the seam real immediately; provider-specific names never escape the adapter.

Keep production types in `src/Notepads/Controls/TextEditor/SyntaxHighlighting/` to avoid a new runtime assembly and keep the diff local to the existing editor. Add no members to `ITextEditor` unless a later consumer demonstrably needs them. Do not use `Extensions`, `FileType`, `Notepads.Controls`, or file I/O as transport for syntax state.

## 9. Component/data-flow design

Likely components and files are:

| Component | Responsibility | Likely file |
| --- | --- | --- |
| `SyntaxLanguage` | Syntax identity independent of preview `FileType`; include `None` and only supported values. | `SyntaxHighlighting/SyntaxLanguage.cs` |
| `SyntaxTokenKind` / `SyntaxSpan` | Provider-neutral semantic kind and UTF-16 start/length. Constructors reject negative/out-of-bounds values. | `SyntaxTokenKind.cs`, `SyntaxSpan.cs` |
| `SyntaxLanguageDetector` | Case-insensitive basename-first, then extension mapping; pure and deterministic. | `SyntaxLanguageDetector.cs` |
| `ISyntaxTokenizer` | `Tokenize(string, SyntaxLanguage)` returning neutral spans; no UWP types or editor access. Cancellation may be accepted but callers must not assume a provider can stop mid-parse. | `ISyntaxTokenizer.cs` |
| `ColorCodeSyntaxTokenizer` | Lazy ColorCode repository/parser creation, language/grammar selection, fragment-offset accumulation, nested-scope flattening, semantic mapping. | `ColorCodeSyntaxTokenizer.cs` plus `ColorCodeLanguages/*.cs` for missing grammars |
| `SyntaxPalette` | Resolve semantic kinds to current light/dark/high-contrast `Color`; centralize all colors. | `SyntaxPalette.cs` |
| `SyntaxHighlightingPolicy` | Enabled/recognized/size decision, debounce choice, and provisional measured cutoff. Pure and testable. | `SyntaxHighlightingPolicy.cs` |
| `RichEditBoxSyntaxRenderer` | UI-thread-only reset and range foreground application using the exact snapshot's UTF-16 offsets. | `RichEditBoxSyntaxRenderer.cs` |
| `SyntaxHighlightingSession` | Per-editor lifetime, snapshot capture, debounce, one bounded background parse, cancellation, document/context/palette generations, stale rejection and cached spans. | `SyntaxHighlightingSession.cs` |

Data flow for an edit is:

1. `TextEditorCore.OnTextChanging` updates the existing plain-text cache only for a content change. The integration increments the document generation but does no parsing or XAML work there.
2. After `TextChanged`, `SyntaxHighlightingSession` captures `TextEditorCore.GetText()` on the UI thread, coalesces edits, and evaluates enabled/language/size policy. Plain/unknown/oversize paths stop here and ensure base formatting is restored once.
3. A bounded worker tokenizes the immutable snapshot. The result carries editor identity, document generation and language/context generation. Cancellation drops queued work; generation checks supersede non-cancellable ColorCode work.
4. The UI dispatcher checks not-disposed, loaded/enabled state, exact generations, current language and snapshot length before applying. Invalid spans are rejected, not clamped.
5. The renderer captures selection start/end/direction and scroll offsets, calls `BatchDisplayUpdates`, resets the full document to the current base foreground, applies coalesced independent `Document.GetRange(start, end)` formats, balances `ApplyDisplayUpdates` in `finally`, and asserts/restores state only if Phase 0 proves restoration itself harmless. It never calls `SetText` or changes `Document.Selection`.

Apply the default foreground before token spans so stale colors cannot survive token deletion, disablement or language change. Coalesce adjacent spans of the same semantic kind and skip spans mapped to base foreground to minimize COM/TOM calls. If full reset is too expensive, optimize only after Phase 5 evidence; do not introduce a partial-state algorithm in v1.

## 10. Language-detection strategy

Keep syntax language separate from preview/content `FileType`. Detection is pure, case-insensitive, basename-first, then longest/specific extension. It reads only `EditingFile.Name` or `EditingFileNamePlaceholder`; it never sniffs contents in v1. Shebang/modeline detection and manual override are deferred because they complicate unsaved/session behavior.

| Language | Initial mappings | Provider plan |
| --- | --- | --- |
| JSON / JSONC | `.json`; `.jsonc`, common config names after corpus selection | built-in JSON; isolated ColorCode grammar for comments |
| Python | `.py`, `.pyw` | built-in |
| PowerShell | `.ps1`, `.psm1`, `.psd1` | built-in |
| YAML | `.yaml`, `.yml` | isolated Notepads-owned ColorCode grammar |
| XML / HTML / CSS | `.xml`, `.xaml`; `.html`, `.htm`; `.css` | built-in |
| JavaScript / TypeScript | `.js`, `.mjs`, `.cjs`; `.ts`, `.tsx` | built-in, subject to TSX corpus correctness |
| shell | `.sh`, `.bash`, `.zsh`, `.profile`, basename `.bashrc`/`.zshrc` | isolated grammar; do not claim dialect completeness |
| TOML / INI | `.toml`; `.ini`, `.cfg`, `.conf`, `.editorconfig` | isolated grammars |
| C / C++ / C# | `.c`; `.cc`, `.cpp`, `.cxx`, `.h`, `.hh`, `.hpp`, `.hxx`; `.cs` | C corpus against C++ grammar or a small C grammar; built-in C++/C# |
| build/config basenames | `Dockerfile`, `Makefile`, `.gitignore` | only enable once an explicit Dockerfile/shell, Make, or ignore-pattern grammar exists |

Missing grammars are definitions consumed by the same ColorCode parser adapter, not editor-specific renderers. Phase 1 must either implement and corpus-test all languages promised for v1 or explicitly narrow the release list; a mapping without a tokenizer returns `None` and stays plain. Filename context is recomputed from `UpdateDocumentInfo` after open, Save As, `RenameAsync`, external reload and session restore. A changed language increments context generation, cancels pending work, clears old colors, then schedules the new language.

## 11. Theme design

Use the smallest semantic set supported reliably across grammars: `Comment`, `Keyword`, `String`, `Number`, `Type`, `Function`, `Property`, `Operator`, `Punctuation`, `Preprocessor`, and `Constant`; everything else is `PlainText`. ColorCode scope-to-kind mapping lives only in its adapter. `SyntaxPalette` owns one cohesive light palette and one dark palette and resolves the base foreground from the same resources as `TextEditorCore.xaml`; no rendering code contains RGB literals.

Theme changes increment palette generation and recolor cached spans without reparsing when document/context generations still match. Listen to `TextEditorCore.ActualThemeChanged` as well as `ThemeSettingsService.OnThemeChanged` so Windows-driven/high-contrast changes are not missed. High contrast is a distinct policy: resolve only documented system theme resources and collapse categories to the system text foreground when adequate category contrast cannot be guaranteed. Accessibility takes precedence over visible token differentiation. Phase 4 must test all Windows high-contrast schemes available on the workstation before enabling differentiated high-contrast colors.

## 12. Performance/large-file strategy

No ColorCode type, grammar, parser, worker or palette is initialized until highlighting is enabled and a recognized file below policy is loaded. A `.txt`, unknown or disabled session pays one filename lookup and no task allocation. Initial recognized-file parsing begins only after load/session restore settles. Content edits use a provisional 250 ms debounce, explicitly tuned in Phase 5; large paste cancels/coalesces just like typing.

Parsing runs off the UI thread; all document/range operations remain on it. Use one app-wide `SemaphoreSlim(1,1)` worker (owned by a small static scheduler inside the module) so background regex jobs from many tabs cannot saturate CPU. Cancellation removes delayed/queued jobs. Since ColorCode cannot abort an active regex, generation checks discard completion, and newly active work waits rather than running concurrently. Only loaded/active editor sessions request work; unload/tab switch cancels pending debounce and application, while a returning tab schedules one current generation.

Start implementation with a deliberately provisional 500 KiB UTF-16 snapshot limit, near the requested 500 KB benchmark point and below the app's normal file-open ceiling; it is a safety setting in code, not a claimed acceptance number. Phase 5 may lower or raise it only from measurements. Above the final cutoff, remain plain text with no opt-in or visible-region mode in v1. Benchmark 1 MB and several-MB files by bypassing the highlighter policy in instrumentation only, never by automatically exposing users to that work. Visible-region/incremental coloring is deferred because ColorCode exposes no resumable lexical state; correct whole-document parsing below a cutoff is simpler and predictable.

## 13. Undo/redo and plain-text integrity strategy

Never use `Document.SetText`, selection-based formatting, undo-buffer clearing, `UndoLimit` toggling, or `BeginUndoGroup` as a coloring workaround. Direct independent-range formatting is allowed only after `REB-1` proves it does not enter or disturb native undo. Every manual undo script starts from a fresh document and records `CanUndo`/`CanRedo`, content and selection before edit, after edit, after highlight, after one undo and after one redo.

Plain-text validation records hashes and disk bytes before/after color, recolor, disable and reopen for UTF-8 with/without BOM, UTF-16 LE/BE where supported, CRLF/LF/CR, supplementary Unicode, combining text and final-newline variants. Assert `GetText(None)`, `TextEditorCore` cache, `LastSavedSnapshot`, session snapshot, dirty flag and plain clipboard data are identical. Notepads' `Copying` handler already constructs a `DataPackage` from `Selection.Text`, so the expected product behavior is plain text; test keyboard, context-menu and cut paths. Do not serialize colors or spans.

## 14. Concurrency/lifecycle model

Each session owns a cancellation source, monotonically increasing document generation, context generation, palette generation, cached immutable spans, loaded/active flag and disposed flag. All transitions occur on the UI thread:

| Event | Required transition |
| --- | --- |
| content `TextChanging` | increment document generation; cancel debounce/queued work; invalidate cached spans |
| post-load/reload/session reset | increment document generation after final text is installed; detect and schedule once |
| Save As/rename/extension change | increment context generation; redetect; clear/schedule without changing document generation |
| external reload | invalidate document and context before reinitialization; schedule only final loaded state |
| `ThemeSettingsService` or `ActualThemeChanged` | increment palette generation; reuse current spans only if document/context match |
| setting disabled | cancel, increment context generation, clear to current base foreground; no parser objects created later |
| tab unload/switch away | cancel delayed/queued/apply work; retain spans only as an optional bounded cache |
| close/`Dispose` | set disposed first, cancel, detach editor/settings/theme handlers, release spans; callbacks become no-ops |

Every parse result carries editor instance identity plus all relevant generations. Completion marshals to that editor's dispatcher and verifies identity, not disposed, loaded/active, enabled, language, document generation and snapshot length before touching the document. A second check occurs immediately before formatting. Exceptions after disposal/cancellation are observed and logged in the existing diagnostic style, never surfaced from `async void`. IME composition is an explicit session state: content generations may advance, but formatting application is deferred until composition ends and then only the newest snapshot is scheduled.

## 15. Detailed phased implementation plan

### Phase 0 — Baseline and technical qualification

**Objective:** resolve whether native range coloring and ColorCode are safe enough to justify production work.

**Likely files/components:** temporary instrumentation in `Controls/TextEditor/TextEditorCore.cs` and `TextEditor.xaml.cs`, or a disposable UWP harness using those exact files; an uncommitted ColorCode adapter spike; a results note under `docs/plans/` only if retained. No product code survives the phase.

**Tasks:** rerun baseline Debug x64; exercise the complete `REB-1..4` matrix; parse real samples through `ColorCode.Core` in Debug and packaged Production x64; launch that package to prove AOT execution, not only compilation; measure cold/warm parser and range-application behavior; test missing grammar feasibility; record event/undo/selection/scroll/IME observations. The completed planning build already establishes that existing ColorCode 2.0.15 can pass Production x64 compilation, but parser execution remains open.

**Validation/completion:** all hard RichEdit gates pass; CC-1 execution passes; CC-2 data is recorded; an explicit decision selects A, a deliberately narrower B, or no-go. Compile success alone is insufficient.

**Risks:** undocumented native undo/input behavior may kill the design; instrumentation can accidentally perturb event ordering. Test actual `TextEditorCore` and remove instrumentation before interpreting the final tree.

**Rollback point/dependency:** starts from baseline commit and depends on no feature work. Revert/discard every spike; a no-go leaves only documentation and the unchanged app.

### Phase 1 — Core abstractions and language coverage

**Objective:** create provider-neutral, UI-free language/token/policy logic and prove advertised mappings/grammars.

**Likely files/components:** new `Controls/TextEditor/SyntaxHighlighting/{SyntaxLanguage,SyntaxTokenKind,SyntaxSpan,SyntaxLanguageDetector,ISyntaxTokenizer,ColorCodeSyntaxTokenizer,SyntaxHighlightingPolicy}.cs`; optional `ColorCodeLanguages/*.cs`; `Notepads.csproj` compile/package reference entries; `tests/Notepads.SyntaxHighlighting.Tests/` as a small `dotnet test` project that links the pure source files rather than requiring a new production assembly.

**Tasks:** implement basename/extension mapping; define strict span invariants and nested-scope precedence; lazily construct the ColorCode parser; accumulate callback fragment offsets to exact UTF-16 positions; map scopes to semantic kinds; add tested grammar definitions for ColorCode gaps or narrow the advertised language list. Keep all Windows/XAML types out of the pure files so the lightweight test project can compile the exact sources. Use the repository's pinned ColorCode 2.0.15; do not upgrade it.

**Validation/completion:** automated tests cover case-insensitive filenames, multi-dot names, unknown/`.txt`, all requested special basenames, CR offsets, surrogate pairs, nested/overlap normalization, malformed/incomplete samples, multiline comments/strings, policy cutoff boundaries and a fake-tokenizer stale-generation evaluator. Every enabled language has a corpus; solution Debug x64 and Production x64 build. No editor is colored yet.

**Risks:** source-link tests can drift if pure files acquire UWP dependencies; custom regex grammars can be incomplete or pathological. Enforce UI-free locality and corpus/timeout benchmarks.

**Rollback point/dependency:** depends on Phase 0 go. One commit adds the neutral model/detector/tests; a second adds the selected adapter/grammars. Either can be reverted without touching editor behavior.

### Phase 2 — Static rendering proof

**Objective:** safely color one already-loaded, recognized document once, with no live-edit scheduling.

**Likely files/components:** new `RichEditBoxSyntaxRenderer.cs` and a temporary/manual invocation owned by `TextEditor.xaml.cs`; `SyntaxPalette.cs`; renderer tests remain manual because they require packaged UWP UI.

**Tasks:** apply current base foreground plus coalesced spans through independent `Document.GetRange`; balance display batching; validate bounds; capture/assert selection and scroll; clear on unrecognized/oversize/disabled input. No parser call or range application occurs in `TextChanging`.

**Validation/completion:** repeat the Phase 0 one-character undo/redo test with the production renderer; exact plain document/cache/clipboard/session/save bytes; no caret, directional-selection or viewport movement; visual correctness for representative static files; event counts understood; Debug and packaged Production x64 run. The phase is incomplete if only colors look correct.

**Risks:** full foreground reset or thousands of TOM calls may be slow; insertion formatting can leak. Stop rather than masking undo/input failures.

**Rollback point/dependency:** depends on Phase 1 and all REB gates. Revert the renderer/integration commit to return to pure unused abstractions.

### Phase 3 — Editing and lifecycle

**Objective:** make highlighting safe during normal editing, tabs and file identity changes.

**Likely files/components:** new `SyntaxHighlightingSession.cs`; narrow hooks in `TextEditor.xaml.cs`, `TextEditorCore.cs` only if a post-content/IME signal is not already exposed, and possibly `ITextEditor.cs` only when proven necessary; no change to `Extensions`.

**Tasks:** add 250 ms coalescing, global single-worker scheduling, cancellation and document/context generations; snapshot on UI, parse background, apply UI; suppress stale completion; defer through IME; hook `Init`, `Reload`, `ResetEditorState`, `UpdateDocumentInfo`, `RenameAsync`, save-to-new-file, Loaded/Unloaded, diff/preview mode as needed, and `Dispose`. Ensure find/replace, whole-document transforms and external reload result in one final generation.

**Validation/completion:** rapid typing/deletion/paste/replace; edit-then-undo/redo; switch/close/reopen tabs during parse; Save As between extensions; rename while pending; external reload; session restore; preview/diff enter/exit; app restart; forced parser exception; no stale/wrong/disposed callback updates and no unobserved tasks. Automated state-machine tests exercise stale rejection with a controllable fake tokenizer. Debug and Production x64 packaged runs pass.

**Risks:** ColorCode work cannot be interrupted; background completion ordering and `async void` event handlers are hazards; composition events may be unavailable at the desired abstraction. Prefer generation discard and explicit task observation; pause for design review if reliable IME state cannot be observed.

**Rollback point/dependency:** depends on stable Phase 2. Revert the session/lifecycle commit to retain static proof only.

### Phase 4 — Settings and themes

**Objective:** deliver minimal automatic opt-out behavior and runtime-safe recoloring.

**Likely files/components:** `Services/AppSettingsService.cs`, `Settings/SettingsKey.cs`, `Views/Settings/TextAndEditorSettingsPage.xaml`/`.xaml.cs`, `Strings/en-US/Settings.resw` and the repository's localization workflow, `SyntaxPalette.cs`, `TextEditor.xaml.cs` theme/settings subscriptions.

**Tasks:** add default-on `IsSyntaxHighlightingEnabled` plus change event and one settings toggle; disable cancels and clears formats; enable schedules only recognized/below-limit active files; define centralized light/dark palettes; use system-resource/fallback policy for high contrast; reuse cached neutral spans on palette-only changes. Do not add a language selector.

**Validation/completion:** setting persists across restart; off means no parser initialization and base formatting; toggle/theme change during parsing cannot apply stale colors; light/dark colors are legible; all available high-contrast schemes preserve text readability; selection/accent colors remain usable; localization resource fallback/build is verified. Full integrity and undo tests repeat after clear/recolor.

**Risks:** explicit character colors can override theme resources; adding English-only resources may regress localization quality. Follow the existing resource workflow and fall back to base foreground in high contrast.

**Rollback point/dependency:** depends on lifecycle safety. Revert the settings/theme commit; core automatic feature can be kept disabled by default during rollback if investigation is needed.

### Phase 5 — Performance and large-file hardening

**Objective:** derive the shipping cutoff/debounce from measurements and protect plain-text responsiveness.

**Likely files/components:** `SyntaxHighlightingPolicy.cs`, scheduler/session/renderer optimizations, checked-in non-copyright benchmark generators or hashes under `tests/fixtures/syntax/`, and a results document under `docs/benchmarks/`.

**Tasks:** run the Section 17 protocol; separate detection, cold grammar construction, warm parse, span normalization and UI application; profile allocation and range count; test rapid supersession and huge paste; coalesce spans and avoid redundant same-color calls; tune debounce/cutoff. Keep whole-document parsing and hard-disable above threshold unless evidence justifies a simpler safe improvement.

**Validation/completion:** measurements demonstrate no parser/highlighter initialization for startup with blank/`.txt`; final cutoff and debounce are recorded with rationale; recognized files at or below cutoff remain responsive; above-cutoff files remain plain and typing does not queue parsing; memory returns after tab close. Compare against the baseline build on the same machine. No fabricated universal milliseconds are required, but any observed regression must be explicitly accepted or fixed.

**Risks:** UI formatting, not tokenization, may dominate; non-cancellable regex may burn CPU after supersession. Lower the cutoff/narrow language coverage before adding incremental complexity.

**Rollback point/dependency:** depends on complete lifecycle/theme behavior. Revert optimizations independently; policy can be made more conservative without changing architecture.

### Phase 6 — Broad regression and release readiness

**Objective:** establish that syntax coloring has not weakened Notepads behavior or packaging.

**Likely files/components:** tests/fixtures, plan/benchmark/user documentation, project/CI validation only; no opportunistic refactor.

**Tasks:** run Section 16 in all advertised languages; build Debug, Release and Production for x86/x64/ARM64 per `AGENTS.md`; install/run packaged Production on available architecture; compare dependency/package size and startup; inspect final diff for unrelated changes; document threshold and unsupported languages.

**Validation/completion:** every advertised mapping passes corpus and manual matrix; ordinary text, large files, IME/RTL, undo and byte integrity pass; known baseline warnings remain distinguished; package installs/launches; all handlers/tasks dispose cleanly. A release is not ready on compile evidence alone.

**Risks:** architecture-specific AOT/native behavior and localization/accessibility regressions can surface late. Remove a failing language or disable the feature rather than waive hard integrity gates.

**Rollback point/dependency:** depends on Phases 0–5. Tag the last validated phase checkpoint; each logical commit is independently revertible, and the global switch provides an operational off path but is not a substitute for fixing integrity defects.

## 16. Validation matrix

| Area | Required cases | Pass evidence |
| --- | --- | --- |
| language/token logic | JSON/JSONC, Python, PowerShell, YAML, XML, HTML, CSS, JS, TS, shell, TOML, INI, C, C++, C#; Dockerfile/Makefile/`.gitignore`/`.editorconfig` only when supported; complete, incomplete and malformed text; multiline strings/comments | expected `(start,length,kind)` fixtures; bounds/non-overlap invariants; unsupported stays plain |
| editing | type inside/at token boundaries, delete/backspace, multiline paste, cut/copy/paste, find/replace one/all, indent/move/join/transform commands | text and dirty state correct; newest generation alone applies; no visible inherited stale color |
| undo/redo | one-character scenario; paste; replace; redo then edit; highlight/recolor/disable between edit and undo | each Ctrl+Z/Ctrl+Y affects only the user's expected text operation; redo is not cleared by coloring |
| selection/input | forward/backward selection, caret at start/end, scroll deep in file, mouse/touch context actions, IME composition, RTL, emoji/surrogates, combining marks | positions/direction/viewport unchanged; composition/candidate behavior normal; spans align to glyph text |
| files | open, ordinary save, Save As across recognized/unknown extensions, rename, external reload accept/decline, encoding/BOM and CR/LF/CRLF, final newline | detection follows final name; exact expected bytes; no derived syntax state persisted |
| lifecycle/modes | restore session, app restart, multiple tabs, switch/close while delayed/parsing/applying, preview, diff, dispose, exception | no wrong-tab/stale/disposed application, leak, crash or unobserved exception; modes retain existing behavior |
| appearance | enable/disable, light/dark switch during parse, Windows high contrast, accent/selection, malformed syntax | legible cohesive colors; clearing uses current base foreground; no reopen required |
| scale | approximately 10 KB, 100 KB, 500 KB, 1 MB, several MB; rapid typing and large paste | metrics recorded; final cutoff enforced; above cutoff causes no automatic parse |
| packaging | Debug/Release/Production, x86/x64/ARM64 builds; packaged Production execution where hardware permits | baseline warnings separated; restore/build/package/run result recorded |

Automate pure rows in `tests/Notepads.SyntaxHighlighting.Tests`; keep RichEditBox, IME, clipboard, theme and packaged-AOT checks as an explicit manual script. UI automation is optional only if it reuses existing tooling; do not introduce a heavyweight framework merely to drive the control.

## 17. Benchmark methodology

Use generated or redistributable fixed corpora near 10 KB, 100 KB, 500 KB, 1 MB and several MB for at least a dense-token format (JSON), a multiline grammar (Python/PowerShell), and a markup format (XML). Record generator seed, exact byte/UTF-16 lengths and SHA-256 so runs are comparable without committing huge fixtures.

On one workstation and power mode, compare baseline commit, feature with setting off, feature opening `.txt`, cold first recognized open, and warm recognized open. Restart the process between cold samples; run at least five iterations and retain every observation rather than only the best. Instrument with `Stopwatch` around detection, grammar initialization, tokenization, normalization, dispatcher wait and range application. Record span/range counts, process CPU time, peak and settled private working set from Visual Studio Diagnostic Tools or Windows Performance Recorder, and GC allocations where the runtime tooling supports them. Use a simple keystroke/paste script or timestamped event instrumentation for edit-to-render latency and count discarded generations/continued stale parses.

Measure: launch to first interactive blank editor; file open to editable text and to final highlight; cold/warm parse; UI application; memory delta after open and after tab close; 30 seconds of rapid typing; 100 KB paste; theme recolor; and behavior above cutoff. The acceptance statement is evidence-based: plain startup/`.txt` must show no highlighter initialization and no repeatable material regression beyond run noise; code editing must remain subjectively responsive and free of long UI-thread stalls; above-cutoff work must be zero. Phase 5 records actual distributions and chooses numeric product thresholds—this plan deliberately does not fabricate them.

## 18. Git/commit strategy

Create `feature/syntax-highlighting` from the planning baseline only after this plan is approved. Suggested coherent checkpoints are:

1. `docs: record syntax highlighting qualification results`
2. `feat: add syntax language and span model`
3. `feat: adapt ColorCode tokenization` (plus separately reviewable missing grammars)
4. `feat: render static syntax spans in RichEditBox`
5. `feat: schedule syntax highlighting across editor lifecycle`
6. `feat: add syntax highlighting setting and palettes`
7. `perf: enforce measured syntax highlighting limits`
8. `test: complete syntax highlighting regression fixtures`

Each commit/phase must build and include its validation record; qualification spikes themselves are discarded. Do not mix dependency upgrades, generated build output, translations unrelated to the setting, formatting churn, or unrelated cleanup. Before each checkpoint, compare against `master`/upstream, inspect `git diff --check`, and ensure the commit can be reverted without hidden dependencies. Rebase/merge decisions remain user-controlled; never force-push shared work.

## 19. Known risks and unresolved questions

- RichEditBox foreground formatting may be undoable or event-producing; this is the primary architectural blocker.
- IME composition and insertion-format inheritance are undocumented and may require deferring application.
- Many range updates may monopolize the UI thread even when parsing is fast.
- ColorCode cannot cancel in-flight regex work, has fragment/string allocation, and lacks requested languages.
- ColorCode grammar correctness for incomplete code and multiline edge cases is uneven and unbenchmarked.
- ColorCode Release/Production .NET Native execution through the parser API is not yet proven.
- Theme clearing must not leave stale explicit colors or defeat high contrast.
- Default RichEditBox copy includes RTF; decide whether presentation color on the rich clipboard is acceptable while preserving exact plain text.
- A safe large-file cutoff is intentionally unresolved until measured.

## 20. Explicit go/no-go gates

1. **RichEdit undo:** one Ctrl+Z after one edit plus coloring undoes only the edit. Failure without a supported non-destructive workaround is no-go for in-place rendering.
2. **Content integrity:** event, dirty-state, clipboard plain text, session text, and saved bytes are unchanged by color/recolor/clear. Any unexplained mutation is no-go.
3. **Selection/IME/viewport:** static and typing stress preserve caret/selection/scroll and defer safely through composition. Repeatable corruption is no-go.
4. **ColorCode compatibility:** Debug and packaged Release/Production .NET Native restore, compile, and execute parser paths without runtime-directive failures. Failure selects another tokenizer or narrows the approach.
5. **Performance:** measured plain-text startup has no highlighter initialization; code-file parse/apply and typing remain responsive at accepted sizes; above a measured threshold highlighting stays off. Failure requires a lower cutoff, narrower scope, or no-go.
6. **Coverage honesty:** every advertised language passes the corpus; unsupported mappings remain plain. Do not ship guessed aliases as support.
7. **Upstream fit:** no editor replacement, extension-system repurposing, unrelated dependency upgrade, or broad refactor is required. If required, stop for an explicit architecture decision.
