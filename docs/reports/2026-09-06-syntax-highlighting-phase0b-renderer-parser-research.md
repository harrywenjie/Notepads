# Syntax Highlighting Phase 0B: Renderer and Parser Research

Status: completed qualification. This report separates Microsoft-documented contracts, runtime observations, and engineering inferences. All product instrumentation described here was disposable and was removed before commit.

## 1. Executive conclusion

No safe and efficient renderer was found. On the actual Notepads `TextEditorCore`, packaged Debug and Production x64 both returned `E_NOINTERFACE` for classic TOM `ITextDocument` and `ITextDocument2`. Therefore `tomApplyTmp`, native undo suspension, and `Freeze`/`Unfreeze` are inaccessible, not merely unqualified. A supported `TextHighlighter` overlay avoided editor mutation, but exact layout parity is impossible for Notepads tabs because `TextEditorCore` sets `Document.DefaultTabStop` while `TextBlock`/`RichTextBlock` exposes no equivalent. Forced visible rendering was also far too slow: about 1.28 seconds for 100 KiB/9,560 ranges and 11.13 seconds for 500 KiB/20,000 ranges in packaged Debug.

Renderer decision: **NO-GO for feature development on the current UWP `RichEditBox` architecture**. Replacing the editor remains out of scope. Parser research remains independently useful, but Phase 1 must not begin until a supported presentation surface is identified (most plausibly during a future platform/editor migration).

## 2. Baseline

The branch began clean at `741d62df63693e9fd84ae445cb8d3df78340aff9` on `feature/syntax-highlighting`. `origin` remained `https://github.com/harrywenjie/Notepads.git`; `upstream` remained `https://github.com/0x7c13/Notepads.git`. No change was made to `master` or either remote.

Tests ran on Windows 11 Pro build 26200, Visual Studio 2022 MSBuild 17.12.36, x64. `Notepads.csproj` targets SDK 22621 with minimum 17763. Untouched Debug x64 and Production x64 solution builds both passed before instrumentation. Phase 0's known `MSB3277` conflicts (`System.Security.Principal.Windows` and `System.Security.AccessControl`, 4.1.1 versus 5.0.0) remained the baseline warning profile; they are not Phase 0B regressions.

Phase 0's public `ITextRange.CharacterFormat.ForegroundColor` result is authoritative: hard **NO-GO** due to undo pollution, event amplification, insertion-format inheritance, viewport movement, and multi-second range rendering. Phase 0B did not retry it as a production candidate.

After all instrumentation and packages were removed, explicit restore/build validation passed again: Debug x64 produced the application/MSIX with no errors; Production x64 completed .NET Native code generation and packaging with **0 errors and 2 warning groups**, exactly the two known `MSB3277` conflicts above. The final source tree matched the starting commit; only this report remained for commit.

## 3. Native TOM research

### 3.1 Two distinct API families share names

`RichEditBox.Document` exposes the WinRT `Windows.UI.Text.ITextDocument`. Microsoft describes `RichEditTextDocument` as an `IInspectable`-based runtime class implementing that interface; its public operations include `Undo()` with no arguments and `BatchDisplayUpdates`/`ApplyDisplayUpdates`. The class API does not expose native TOM's undo-suspension, `Freeze`, `Unfreeze`, or temporary-formatting calls. [RichEditTextDocument](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.richedittextdocument?view=winrt-26100) and [WinRT ITextDocument](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument?view=winrt-26100)

The installed Windows 10 SDK 10.0.22621.0 header `winrt/windows.ui.text.h` confirms that WinRT `Windows.UI.Text.ITextDocument2` has IID `f2311112-8c89-49c9-9118-f057cbb814ee`, derives from `IInspectable`, and adds only `AlignmentIncludesTrailingWhitespace` and `IgnoreTrailingCharacterSpacing`. Microsoft's generated header identifies it as part of `RichEditTextDocument`. It is **not** native TOM 2. [Microsoft-generated WinRT header](https://github.com/microsoft/win32metadata/blob/main/generation/WinSDK/RecompiledIdlHeaders/winrt/windows.ui.text.h#L2116)

Native TOM is classic COM in `tom.h`:

- native `::ITextDocument`: IID `8cc497c0-a1df-11ce-8098-00aa0047be5d` ([Microsoft TOM GUID documentation](https://learn.microsoft.com/en-us/windows/win32/controls/use-tom-guids));
- native `::ITextDocument2`: IID `c241f5e0-7206-11d8-a2c7-00a0d1d6c6b3` (verified in installed SDK `um/tom.h`);
- the SDK marks both as dual classic COM interfaces. `ITextDocument` derives from `IDispatch`, and native `ITextDocument2` derives from it. Although the Learn summary describes `ITextDocument` as inheriting `IUnknown`, any hand-written vtable must include the intervening `IDispatch` slots from the authoritative SDK declaration.

Microsoft's supported acquisition recipe for native TOM starts from a **desktop rich edit control**: send `EM_GETOLEINTERFACE`, receive `IRichEditOle`, then call `IUnknown::QueryInterface`. The native TOM reference lists “desktop apps only”; TOM 2 also exposes HWND- and IME-oriented methods. Neither page states that a UWP `RichEditTextDocument` implements these native interfaces. [native ITextDocument](https://learn.microsoft.com/en-us/windows/win32/api/tom/nn-tom-itextdocument), [native ITextDocument2](https://learn.microsoft.com/en-us/windows/win32/api/tom/nn-tom-itextdocument2)

### 3.2 ABI acquisition is possible to test, not documented to succeed

All WinRT interfaces require `IInspectable`, which in turn requires `IUnknown`; those bases supply `QueryInterface`, reference counting, and runtime inspection. `IInspectable.GetIids` returns the interfaces implemented by a runtime object, excludes `IUnknown`/`IInspectable`, and guarantees that querying a returned IID succeeds. [WinRT type system](https://learn.microsoft.com/en-us/uwp/winrt-cref/winrt-type-system#iinspectable-and-iunknown), [IInspectable.GetIids](https://learn.microsoft.com/en-us/windows/win32/api/inspectable/nf-inspectable-iinspectable-getiids)

Managed code can obtain an `IUnknown` pointer with `Marshal.GetIUnknownForObject`; both it and a successful `Marshal.QueryInterface` increment reference counts that must be balanced with `Marshal.Release`. The `Marshal.QueryInterface` API lists UWP 10.0 as applicable. This establishes a supported way to *ask* for an IID, not a promise that `RichEditTextDocument` supports native TOM. [GetIUnknownForObject](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.getiunknownforobject), [Marshal.QueryInterface](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.queryinterface)

### 3.3 Relevant native-only primitives

- `ITextDocument::Undo(tomFalse, ...)` suspends undo processing and `Undo(tomTrue, ...)` restores it. This is not the projected UWP `Undo()` method. [native Undo](https://learn.microsoft.com/en-us/windows/win32/api/tom/nf-tom-itextdocument-undo)
- `ITextDocument::Freeze` increments a nesting count and disables screen updates while nonzero; `Unfreeze` decrements it and reenables updates at zero. These calls suppress painting, not necessarily layout work. [Freeze](https://learn.microsoft.com/en-us/windows/win32/api/tom/nf-tom-itextdocument-freeze), [Unfreeze](https://learn.microsoft.com/en-us/windows/win32/api/tom/nf-tom-itextdocument-unfreeze)
- `ITextFont::Reset(tomApplyTmp)` means “apply temporary formatting.” The extended edit-style documentation says such temporary formatting is used, for example, by spell checkers for squiggles and can be hidden with `SES_HIDETEMPFORMAT`. [ITextFont.Reset](https://learn.microsoft.com/en-us/windows/win32/api/tom/nf-tom-itextfont-reset), [tomConstants](https://learn.microsoft.com/en-us/windows/win32/api/tom/ne-tom-tomconstants), [SES_HIDETEMPFORMAT](https://learn.microsoft.com/en-us/windows/win32/controls/em-geteditstyleex)

Microsoft documentation does **not** specify for a UWP-hosted RichEdit instance whether `tomApplyTmp` is excluded from undo, saved/clipboard RTF, `TextChanging`/`TextChanged`, dirty state, selection/caret state, IME composition, or all supported OS builds. It also does not document the required sequence for clearing/replacing overlapping temporary foreground colors. Those are runtime gates, not facts to infer from the word “temporary.”

### 3.4 Threading and lifetime

`RichEditTextDocument` is marked agile, but Microsoft explicitly warns that agility does not imply thread safety and that even agile UI-related objects can require UI-thread calls. Keep document acquisition and every render mutation on the owning editor's UI thread; parse only immutable strings off-thread. [Windows Runtime objects in a multithreaded environment](https://learn.microsoft.com/en-us/windows/apps/develop/threading/winrt-objects-multithreaded), [UWP UI-thread responsiveness](https://learn.microsoft.com/en-us/windows/uwp/debug-test-perf/keep-the-ui-thread-responsive)

Any native range/font pointer is subordinate to its editing instance. Microsoft notes that TOM objects become unusable and return `CO_E_RELEASED` after the associated editing instance is deleted. Do not retain native COM pointers across editor disposal, document replacement, reload, or tab reuse. [About TOM](https://learn.microsoft.com/en-us/windows/win32/controls/about-text-object-model#tom-interface-conventions)

## 4. COM/ABI acquisition results

A disposable harness instantiated the real `TextEditor`, obtained its actual `TextEditorCore.Document`, called `Marshal.GetIUnknownForObject`, queried each IID, immediately released every successful QI pointer, released the initial `IUnknown` in `finally`, disposed the editor, and exited. The harness ran from installed MSIX packages, not an unpackaged test process.

| Interface | IID | Debug x64 | Production x64 .NET Native |
| --- | --- | --- | --- |
| `IUnknown` | `00000000-0000-0000-C000-000000000046` | `S_OK` | `S_OK` |
| `IInspectable` | `AF86E2E0-B12D-4C6A-9C5A-D7AA65101E90` | `S_OK` | `S_OK` |
| WinRT `ITextDocument` | `BEEE4DDB-90B2-408C-A2F6-0A0AC31E33E4` | `S_OK` | `S_OK` |
| WinRT `ITextDocument2` | `F2311112-8C89-49C9-9118-F057CBB814EE` | `S_OK` | `S_OK` |
| WinRT `ITextDocument3` | `75AB03A1-A6F8-441D-AA18-0A851D6E5E3C` | `S_OK` | `S_OK` |
| classic TOM `ITextDocument` | `8CC497C0-A1DF-11CE-8098-00AA0047BE5D` | `0x80004002 E_NOINTERFACE` | `0x80004002 E_NOINTERFACE` |
| classic TOM `ITextDocument2` | `C241F5E0-7206-11D8-A2C7-00A0D1D6C6B3` | `0x80004002 E_NOINTERFACE` | `0x80004002 E_NOINTERFACE` |

Debug reported runtime type `Windows.UI.Text.RichEditTextDocument`; Production reflection was blocked by .NET Native, but its known WinRT IIDs and HRESULTs matched Debug. Calls ran on the app UI thread. x86/ARM64 runtime tests were unnecessary after x64 proved that the required interface is absent from the exact object; no architecture-independent native renderer can be built on an interface unavailable on the primary target.

## 5. Supportability assessment

| Surface | Microsoft contract | UWP/Store assessment |
| --- | --- | --- |
| `Windows.UI.Text.ITextDocument`, ranges, character formats | Universal API Contract; documented for UWP | Supported, but Phase 0A runtime semantics still determine fitness |
| `BatchDisplayUpdates` / `ApplyDisplayUpdates` | Public WinRT document API | Supported batching; no documented undo suppression |
| `TextHighlighter` on `TextBlock`/`RichTextBlock` | Universal API Contract v5 (16299) | Supported presentation API; not available on `RichEditBox` |
| `Marshal.QueryInterface` | API page lists UWP 10.0 | Supported mechanism to query an IID; does not expand the queried object's contract |
| native TOM `ITextDocument*`, `ITextFont*` | Win32 pages say desktop apps only | Undocumented/uncontracted for this UWP host, even if QI succeeds |
| Win2D `CanvasTextLayout` overlay | Microsoft Win2D UWP API | Supported graphics route; independent editor layout and accessibility remain app responsibilities |

.NET Native documentation says most P/Invoke and COM interop scenarios and WinRT marshaling remain supported, including `IUnknown`, but it also says COM objects cannot be called through `IDispatch`. Native TOM's SDK declarations are dual `IDispatch` interfaces. A manually declared early-bound/vtable projection might avoid Automation dispatch, but Microsoft does not document that combination for UWP or guarantee it under .NET Native. Therefore Debug/JIT success is insufficient: Production `.NET Native` compile, install, runtime, and architecture tests are mandatory. [Microsoft .NET Native interop differences](https://learn.microsoft.com/en-us/windows/uwp/dotnet-native/migrating-your-windows-store-app-to-net-native#interop-differences)

Store support is also unresolved. The Windows App Certification Kit requires UWP packages to use WinRT or supported Win32 APIs and checks native imports plus the managed approved profile. Direct QI may not create an obvious forbidden import, so WACK passing would be useful but would not turn an unpublished interface implementation into a compatibility promise. [WACK supported-API test](https://learn.microsoft.com/en-us/windows/uwp/debug-test-perf/windows-app-certification-kit-tests#supported-api-test)

Supportability conclusion: native TOM is **INACCESSIBLE** on the tested actual editor in both required packaged configurations. Even a hypothetical private acquisition route would remain outside the UWP contract and unacceptable for a Store-quality, upstream-friendly application. No further private COM/window-handle techniques were attempted.

Architectures and Windows servicing remain risks. COM IIDs are stable, but there is no published commitment that UWP RichEdit exposes native TOM on x86, x64, ARM64, every supported Windows version, or future builds. Manually declared vtables must exactly match SDK order and native widths; each packaged architecture must compile, and available architectures must be exercised.

## 6. tomApplyTmp results

**NOT RUN — INACCESSIBLE.** `tomApplyTmp` requires classic TOM range/font interfaces. Because both classic document QIs returned `E_NOINTERFACE`, there was no valid range acquisition path. Calling a guessed vtable or using a private implementation trick would invalidate the supportability gate.

## 7. undo-suspension results

**NOT RUN — INACCESSIBLE.** `Undo(tomFalse)`/`Undo(tomTrue)` is the classic TOM signature; UWP exposes only parameterless `Undo()`. There is no supported suspension call to test. `ClearUndoRedoHistory` remains explicitly disallowed because it destroys user history. [ClearUndoRedoHistory](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.richedittextdocument.clearundoredohistory)

## 8. Freeze/Unfreeze results

**NOT RUN — INACCESSIBLE.** Native `Freeze`/`Unfreeze` is not present on WinRT `ITextDocument`, and the classic interface could not be acquired. Phase 0 already showed that the supported `BatchDisplayUpdates` pair did not suppress per-range events or make ordinary range formatting acceptable.

## 9. Native renderer benchmarks

No native-renderer benchmark exists because there is no callable native renderer. Benchmarking a standalone desktop RichEdit would not answer the question and was intentionally rejected. The “best native” result is therefore **none / inaccessible**, not zero milliseconds.

## 10. TextHighlighter/overlay research

### 10.1 Supported API surface

`Windows.UI.Xaml.Documents.TextHighlighter` applies foreground/background brushes to one or more `TextRange` values. Its ranges are measured as a Unicode-character start index and length. It was introduced in Universal API Contract v5 / Windows 10 16299. [TextHighlighter](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.documents.texthighlighter?view=winrt-26100), [XAML TextRange](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.documents.textrange?view=winrt-26100)

Microsoft exposes `TextHighlighters` collections on `TextBlock` and `RichTextBlock`, but not on `RichEditBox`. Thus an overlay must be a second read-only text control; it cannot decorate the editor's own glyph runs. [TextBlock.TextHighlighters](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.textblock.texthighlighters?view=winrt-26100), [RichTextBlock.TextHighlighters](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richtextblock.texthighlighters?view=winrt-26100), [RichEditBox API](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richeditbox?view=winrt-26100)

`TextBlock` is Microsoft's lightweight read-only control and is generally faster than `RichTextBlock`, but its documentation says it is intended for small amounts of text and designed as a single paragraph. `RichTextBlock` is read-only and supports paragraph/inline layout at greater complexity. Neither is documented as sharing RichEdit's layout engine. [TextBlock](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.textblock?view=winrt-26100), [RichTextBlock](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.richtextblock?view=winrt-26100)

### 10.2 Integrity benefit and alignment cost

Because highlight ranges live in the separate display control, this route does not mutate `RichEditBox.Document`, its undo stack, saved bytes, or clipboard source. That is a structural benefit, but only if the original `RichEditBox` remains the sole input/selection/accessibility control.

There is no Microsoft contract guaranteeing equivalent glyph positions between RichEdit and either XAML display control. A prototype must synchronize at least font family/size/stretch/style/weight, character spacing, language, flow direction/reading order, wrapping width, alignment, line height/stacking, padding, tab stops, zoom/text scaling, and both scroll axes. RichEdit also has document paragraph/default-tab state and an internal control template/`ScrollViewer`; the display controls expose a different property set. Font fallback, bidi shaping, tabs, extremely long lines, CR/LF normalization, DPI, and accessibility text scaling are mandatory adversarial cases.

Public UWP `ITextRange.GetPoint` and `GetRect` can retrieve screen coordinates/bounds and may help diagnose or anchor visible ranges, but a bounding rectangle does not expose the editor's complete shaped-glyph run or all wrapped fragments. Per-token geometry queries would also add UI-thread/COM cost. [GetPoint](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange.getpoint), [GetRect](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange.getrect)

### 10.3 Input and accessibility constraints

The overlay must be non-hit-testable and must never take focus; the RichEditBox must remain responsible for pointer selection, caret, keyboard, touch/pen, clipboard, and IME composition. Microsoft says apps normally need not interact with the IME directly, but explicitly requires end-to-end testing of the candidate window/composition experience. An overlay that obscures composition underlines or candidate text fails the gate. [Microsoft IME guidance](https://learn.microsoft.com/en-us/windows/apps/develop/input/input-method-editors)

RichEditBox's automation peer reports the Edit control type and supports UI Automation `TextPattern`. A duplicate TextBlock/RichTextBlock would report read-only text, so set `AutomationProperties.AccessibilityView="Raw"` on the overlay to remove it from Control/Content views, then verify Narrator and Accessibility Insights. `Raw` does not remove it from the raw tree. [RichEditBoxAutomationPeer](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.automation.peers.richeditboxautomationpeer?view=winrt-26100), [AccessibilityView](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.automation.peers.accessibilityview?view=winrt-26100), [Microsoft redundant-element guidance](https://learn.microsoft.com/en-us/accessibility-tools-docs/items/uwpxaml/control_iscontrolelement)

All overlay creation, text assignment, range updates, layout-property reads, and scrolling transforms belong on the owning UI thread. Parsing can remain off-thread over a captured immutable string. Microsoft notes that XAML `DependencyObject` instances have UI-thread affinity and only `Dispatcher` may be accessed directly from another thread. [DependencyObject.Dispatcher](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.dependencyobject.dispatcher?view=winrt-26100)

### 10.4 Unspecified behavior

Microsoft documentation does not define overlapping-highlighter precedence, mutation cost for many ranges, virtualization, partial invalidation behavior, or performance on editor-sized documents. It also does not promise that a transparent base foreground plus colored ranges avoids duplicate antialiasing when layered over visible RichEdit glyphs. These require direct measurement and visual comparison; API availability alone is not qualification.

## 11. Overlay alignment results

The static layout gate failed before an overlay editor was justified. Notepads computes a font-dependent tab width and assigns it to `TextEditorCore.Document.DefaultTabStop`; the installed SDK exposes that property only on `Windows.UI.Text.ITextDocument`. Neither `TextBlock` nor `RichTextBlock` has a public equivalent. Consequently the two controls cannot be guaranteed to position text following a tab identically. Notepads also sets RichEdit paragraph spacing through `ITextParagraphFormat.SetLineSpacing(Exactly, FontSize)`, while the display controls use their separate XAML line-layout properties.

This is a correctness failure for a primary source-code use case, not a cosmetic risk. Other unmatched states—RichEdit's internal scroll extent, CR normalization, wrapping fragments, font fallback, emoji, combining marks, bidi, selection/caret/IME adorners, zoom, and DPI—would still require a large interactive matrix. Per the task's “only if static research says alignment is plausible” condition, no misleading polished overlay or pixel-diff was built after the tab-stop mismatch proved exact general synchronization impossible. A display layer over or under the editor also has no supported way to preserve every caret, selection, find, and composition visual while hiding only RichEdit's base glyph foreground.

Overlay feasibility: **NO-GO for a whole-document production renderer**. A future visible-line overlay would still need exact layout and input validation and was not designed in this phase.

## 12. Overlay performance results

A disposable packaged Debug x64 harness measured a `TextBlock` with eight semantic-color `TextHighlighter` objects and deterministic realistic-density ranges. `RenderTargetBitmap.RenderAsync` was used to force visible rendering; simple collection mutation or `UpdateLayout` alone understated deferred XAML work. These are single-run qualification measurements, not product acceptance benchmarks:

| Text | Ranges | Add ranges | First synchronous layout | First forced visible render | Replace ranges | Forced render after replace |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 10 KiB | 956 | 0.82 ms | 0.01 ms | 102.38 ms | 0.37 ms | 98.03 ms |
| 100 KiB | 9,560 | 1.88 ms | 82.55 ms | 1,276.18 ms | 1.83 ms | 2,989.91 ms |
| 500 KiB | 20,000 | 4.05 ms | 1,244.71 ms | 11,129.13 ms | 3.91 ms | 27,302.18 ms |

The range collections themselves scale cheaply; layout/rasterization does not. `RenderTargetBitmap` includes off-screen raster/readback cost and is not a direct measurement of the compositor's normal presentation latency. It was used because `UpdateLayout` alone did not settle deferred drawing, so the forced-render column is an auditable upper-bound path rather than a claimed frame time. The benchmark rendered only a 900 x 600 visible element, so it does not overclaim full-document raster cost, scroll smoothness, or memory. An earlier bounded run including 1 MiB and 3 MiB was stopped after the process accumulated about 51 CPU seconds and 676 MiB working set without producing its final result. Because output was emitted only at completion, that aborted run cannot attribute the stall to a specific large case and is recorded only as a safety observation.

At 100 KiB, the best forced render (~1.28 s) is still about 116 times Phase 0's ~11 ms parser result and roughly half the rejected UWP range renderer's ~2.45 s. It fails the objective that rendering be the same order of magnitude as parsing. `RichTextBlock` was not benchmarked: Microsoft positions it as the more capable/heavier control, and it cannot repair the decisive tab/layout mismatch. TextHighlighter classification: **supported but too slow and layout-incompatible for this use**.

## 13. Other renderer alternatives

### 13.1 Win2D/Composition overlay

Win2D is a Microsoft Windows Runtime API for GPU-accelerated Direct2D drawing in UWP. `CanvasTextLayout.SetBrush(start, count, brush)` assigns colors to character ranges, and `DrawTextLayout` renders the result. The layout exposes bidi depth, line metrics, wrapping, alignment, tabs, caret positions, and character regions. [Win2D introduction](https://microsoft.github.io/Win2D/WinUI2/html/Introduction.htm), [CanvasTextLayout.SetBrush](https://microsoft.github.io/Win2D/WinUI2/html/M_Microsoft_Graphics_Canvas_Text_CanvasTextLayout_SetBrush.htm), [CanvasTextLayout properties](https://microsoft.github.io/Win2D/WinUI2/html/Properties_T_Microsoft_Graphics_Canvas_Text_CanvasTextLayout.htm)

This is technically capable of colored text, but `CanvasTextLayout` is still a second, independently shaped layout. It does not consume RichEdit's internal line/glyph layout, selection, caret, IME, clipboard, or automation peer. Matching more knobs does not establish pixel identity. It also makes Notepads responsible for device/resource lifetime, invalidation, clipping, scroll transforms, DPI/text scaling, and accessibility behavior. Windows Composition can efficiently position/clip a rendered surface, but Composition itself does not provide a RichEdit text-decoration hook; pixels still need to come from Win2D/Direct2D. [CanvasDrawingSession.DrawTextLayout](https://microsoft.github.io/Win2D/WinUI2/html/M_Microsoft_Graphics_Canvas_CanvasDrawingSession_DrawTextLayout_2.htm), [CompositionGraphicsDevice](https://learn.microsoft.com/en-us/uwp/api/windows.ui.composition.compositiongraphicsdevice?view=winrt-26100)

Assessment: plausible only as a disposable comparison if TextHighlighter layout fails for performance reasons but its alignment model appears salvageable. It is not a shortcut around alignment or accessibility gates.

### 13.2 XAML `Run`/`Span` trees

Building token-colored `Run`/`Span` elements inside a `TextBlock` or `RichTextBlock` is supported read-only XAML rendering, but it duplicates the document and creates per-token dependency objects/inlines. It retains all overlay alignment and interaction risks and is likely higher-allocation than grouping `TextHighlighter.Ranges` by semantic color. No primary Microsoft source promises suitability for thousands of frequently replaced token nodes, so do not qualify it without benchmarks.

### 13.3 DirectWrite/Direct2D or a custom editor

A custom DirectWrite renderer could own exact glyph shaping for its own surface, but cannot make that surface share RichEdit's undisclosed layout. Achieving exact selection, caret, hit-testing, scrolling, touch/pen, UI Automation, and IME behavior would amount to building or replacing the editor engine. That violates the minimal-divergence goal unless RichEdit is first proven fundamentally unusable and the owner explicitly broadens scope.

### 13.4 Public RichEdit formatting

Public `ITextRange.CharacterFormat.ForegroundColor` is the only supported in-engine coloring route. It shares the editor's exact glyph layout but mutates document character formatting. Its undo, event, dirty-state, clipboard, and serialization behavior is determined by the Phase 0A runtime evidence. Public `BatchDisplayUpdates` can batch display, but the public API offers no undo-suspension or temporary-format flag. [ITextRange](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextrange?view=winrt-26100), [ITextCharacterFormat.ForegroundColor](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextcharacterformat.foregroundcolor), [WinRT ITextDocument](https://learn.microsoft.com/en-us/uwp/api/windows.ui.text.itextdocument?view=winrt-26100)

## 14. ColorCode pathological-regex root causes

Track B inspected the authoritative ColorCode 2.0.15 tag (`e6c2701c365a7a91d74d7ef38a6b60075008ce94`). The failure is in specific built-in grammar expressions, not in the RichEdit renderer.

### JSON

The JSON grammar defines the string expression as:

```regex
"[^"\\]*(?:\\[^\r\n]|[^"\\]*)*"
```

It uses that expression both as the JSON-string rule and inside the JSON-key rule. The trailing alternative `[^"\\]*` can match the empty string and is itself repeated by `*`; ordinary runs can also be partitioned among the leading star and arbitrarily many outer iterations. When a quote starts a long string but no valid closing quote exists, the backtracking engine explores a combinatorial set of partitions. A minimized generated shape was `"` + many `a` characters + a terminal backslash. At only 1,000 UTF-16 code units, the isolated stock rule exceeded a 100 ms regex timeout on .NET 8.0.30. This diagnosis is directly grounded in the [2.0.15 JSON grammar](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Compilation/Languages/Json.cs).

### JavaScript

The JavaScript block-comment rule is:

```regex
/\*([^*]|[\r\n]|(\*+([^*/]|[\r\n])))*\*+/
```

`[^*]` already includes CR and LF, so `[^*]` and `[\r\n]` overlap. Inside the nested branch, `[^*/]` likewise already includes CR and LF. An unterminated comment containing newlines therefore creates duplicate successful paths at each newline, nested inside repetition. A single `/*` followed by CR characters timed out at 100 ms at length 1,000. Repeated `/*\r` also creates many possible unanchored match starts. See the [2.0.15 JavaScript grammar](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Compilation/Languages/JavaScript.cs).

ColorCode amplifies the risk by joining every language rule into one alternation and constructing `new Regex(...)` with no timeout. Parsing is synchronous and advances with `Match`/`NextMatch`; there is no cancellation check. These are source facts from [LanguageCompiler](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Compilation/LanguageCompiler.cs) and [LanguageParser](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Parsing/LanguageParser.cs). The disposable .NET 8 measurements isolate the bad rules; the earlier packaged UWP >10-second results establish that the whole ColorCode path is also affected on the target runtime.

## 15. Hardened-regex results

Two language-equivalent candidate rewrites were tested without modifying ColorCode or Notepads:

```regex
# JSON: every repeated branch consumes input and branches have disjoint prefixes
"(?:\\[^\r\n]|[^"\\])*"

# JavaScript block comment: every iteration consumes one character
/\*(?:[^*]|\*(?!/))*\*/
```

The stock and candidate expressions returned identical match index/length/value sequences for 11 curated complete/incomplete samples and 43,690 exhaustive generated samples over language-specific four-character alphabets through length seven; there were zero observed differences. This is useful equivalence evidence, not a proof for every Unicode string or for the complete combined grammar.

Measurements below are one post-warm-up pass in separate Release `net8.0` processes on this workstation. A timeout result means `RegexMatchTimeoutException`, not an abandoned worker:

| Case | Length | Stock | Candidate |
|---|---:|---:|---:|
| JSON, one unterminated string | 1,000 | >100 ms timeout | not separately needed at this size |
| JSON, one unterminated string | 1,000,000 | not run unbounded | 38.10 ms |
| JavaScript, one unterminated comment with CRs | 1,000 | >100 ms timeout | not separately needed at this size |
| JavaScript, one unterminated comment with CRs | 100,000 | not run unbounded | 4.81 ms |
| JavaScript, one unterminated comment with CRs | 1,000,000 | not run unbounded | 44.87 ms |
| JavaScript, repeated unterminated `/*\r` candidates | 100,000 | >100 ms timeout | **>100 ms timeout** |

The JSON rewrite removes the demonstrated catastrophic ambiguity. It nevertheless allocated 29,360,272 bytes while rejecting the 1 MB input; the simplified JavaScript rule allocated 46,137,176 bytes on the 1 MB single-start case. The JavaScript rewrite removes the within-comment ambiguity, but an unanchored regex still retries from each `/*` candidate and can be quadratic when none terminates. In contrast, count-only single-pass scanners consumed the 1 MB pathological JSON and JavaScript shapes in 0.58 ms and 0.70 ms respectively with 40 measured bytes allocated.

Therefore a grammar-only patch is insufficient protection for arbitrary JavaScript input. A single-pass comment scanner, a regex timeout, and a conservative size policy remain necessary. These numbers characterize .NET 8, not .NET Native; packaged Production must repeat the candidate/timeout tests before adoption.

## 16. Regex-timeout results

The stock compiler uses the infinite-timeout constructor. There is no setting or overload on `LanguageCompiler`. However, [`ILanguageCompiler`](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Compilation/ILanguageCompiler.cs) is public, and [`CompiledLanguage`](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Compilation/CompiledLanguage.cs) has a public constructor accepting a `Regex` and capture map. A host can inject a custom compiler without modifying the package, but it must reproduce ColorCode's private rule-combination and capture-index logic; that is maintained adapter code, not a configuration switch.

A disposable `ILanguageCompiler` returned a `CompiledLanguage` whose isolated stock JSON rule used a 100 ms timeout. `LanguageParser.Parse` propagated `RegexMatchTimeoutException` after 106.42 ms for the 1,000-character pathological input on .NET 8.

The experiment was then repeated through the real ColorCode.Core 2.0.15 parser API in the installed packaged Notepads Production x64 .NET Native process. A custom compiler wrapped the compiled JavaScript grammar in `Regex(..., TimeSpan.FromMilliseconds(100))`; parsing the 110,002-character bounded pathology raised `RegexMatchTimeoutException` after 109.88 ms with zero callbacks. A following normal compiled-regex match completed in 0.04 ms, proving the packaged process remained usable. A direct timed JavaScript regex also timed out at 94.87 ms. Debug x64 produced the same exception path (136.84 ms direct; the end-to-end custom-compiler run was retained only in the final Production harness). Thus the timeout constructor, exception, recovery, ColorCode compiler seam, package, and .NET Native execution gates pass on x64.

The normal compiled-regex recovery match occurred in the same packaged run, not after the end-to-end parser exception specifically. It proves process/runtime recovery, while a future adapter test should reuse the same parser instance after timeout. The absolute elapsed time can be slightly below or above the requested timeout; .NET documents timeout as an approximate maximum for backtracking work, not a wall-clock deadline for construction and exception delivery. The 100 ms value was a qualification probe, not a selected production policy.

Timeout handling must be all-or-nothing. `LanguageParser` invokes callbacks as matches are discovered, so a timeout on a later `NextMatch` can occur after earlier fragments were delivered. The adapter must accumulate neutral spans off-screen, publish only after successful completion, catch timeout as a normal “no highlighting” result, and log/measure it without failing file open or edit. Regex timeouts are containment, not incremental cancellation or a correctness fix.

## 17. Span-only parser results

ColorCode's parser API is renderer-neutral, but not span-only internally: for every unmatched and matched fragment it allocates `Substring` values and new scope lists, builds scope trees, and calls `Action<string, IList<Scope>>`. A consumer can flatten scopes and retain only absolute spans, but cannot avoid those parser allocations. Absolute offsets must be computed by accumulating each callback fragment's UTF-16 `Length`, then recursively adding child-scope offsets. [LanguageParser source](https://github.com/CommunityToolkit/ColorCode-Universal/blob/v2.0.15/ColorCode.Core/Parsing/LanguageParser.cs)

A disposable single-pass JSON lexer emitted value-type `(start, length, kind)` spans directly into a `List<T>`. It recognized strings with escapes, numbers, and `true`/`false`/`null`; an unterminated string consumed to end of snapshot once. On the same dense generated JSON snapshot, after warm-up:

| Length | ColorCode 2.0.15 | ColorCode allocated | Span lexer | Span lexer allocated | Spans/scopes |
|---:|---:|---:|---:|---:|---:|
| 10,000 | 1.90 ms | 1,683,336 B | 0.14 ms | 49,416 B | 1,334 / 1,333 |
| 100,000 | 20.53 ms | 16,835,336 B | 2.02 ms | 393,552 B | 13,334 / 13,333 |
| 500,000 | 77.19 ms | 84,177,080 B | 2.37 ms | 3,146,136 B | 66,667 / 66,666 |
| 1,000,000 | 136.55 ms | 168,355,336 B | 3.19 ms | 6,291,888 B | 133,334 / 133,333 |

The 1 MB proof was about 43 times faster and allocated about 96% less in this synthetic dense case. It is not feature-equivalent: it does not distinguish keys from values, validate strict JSON, or implement every desired language. It does establish that a neutral span-only, linear scanner is feasible and gives cheap cancellation granularity (for example every 4 KiB or line) without substring allocation.

Multiline lexical state must be explicit for any incremental/line parser: block comments, strings, heredocs, and YAML block scalars cannot be correct when lines are parsed independently without carrying prior-line state. ColorCode's nested-language path also needs a dedicated adapter test: the source increments the entire captured-style tree inside a loop over its top-level scopes, which appears capable of applying the offset repeatedly when a nested match returns multiple roots. Treat nested scopes as unresolved until a reproducer verifies absolute spans; do not silently flatten untested nesting.

## 18. Parser alternatives

No additional external library was qualified in Track B. None has both authoritative compatibility evidence and an actual legacy-UWP Debug plus packaged Production .NET Native execution result, so naming a package here would create false confidence.

The two serious parser choices remain behind the neutral parser interface:

1. **Hardened ColorCode adapter:** preserve broad built-in coverage, replace/copy the compiler to impose timeouts, override the known-dangerous grammars, buffer results, reject timeouts/stale generations, and apply strict file-size policy. This carries capture-map and upstream-grammar maintenance and still cannot cooperatively cancel a running regex.
2. **Small stateful span lexers:** deterministic linear scans, low allocation, explicit cancellation, and exact malformed-input behavior for a deliberately small language set. This has the lowest runtime risk but the highest per-language correctness/maintenance burden, especially JavaScript/TypeScript, PowerShell, Bash, and YAML.

The evidence rules out **unmodified ColorCode** for arbitrary files. It does not yet choose between a hardened adapter and custom lexers. The timeout compiler and exception path passed packaged Production x64; the candidate grammar rewrites and span lexer were measured only in the disposable desktop harness and would still require packaged Production repetition. Before any parser implementation, define the v1 language subset and build corpus tests for multiline state and malformed input. A hybrid is permissible only at the replaceable parser boundary and must expose the same neutral UTF-16 span contract.

## 19. End-to-end architecture comparison

| Combination | Integrity/support | Performance | Coverage/maintenance | Decision |
| --- | --- | --- | --- | --- |
| Hardened ColorCode + classic TOM temporary formatting | Would preserve RichEdit layout if available; native interface is absent and desktop-only | Not measurable | Parser maintenance moderate | **NO-GO: renderer inaccessible** |
| Hardened ColorCode + whole-document `TextHighlighter` overlay | Presentation-only and supported, but separate layout cannot match tabs or guarantee IME/caret/selection | ~1.28 s forced render at 100 KiB; ~11.13 s at 500 KiB | Broad selected built-ins; duplicated text/layout | **NO-GO: layout and performance** |
| Small stateful lexers + `TextHighlighter` overlay | Parser improves determinism; renderer defects unchanged | Parser can be sub-millisecond on pathologies; renderer dominates | High per-language lexer burden | **NO-GO: renderer** |
| Parser + Win2D/custom drawing overlay | Supported graphics, but independent shaping, input adornments, accessibility, device lifetime, and scroll sync become app code | Unmeasured; may draw faster but cannot solve parity | High divergence and maintenance | **Not justified** |
| Replace editor engine | Could provide a native decoration API | Unknown | Very high behavior, IME, accessibility, packaging, and rebase risk | **Out of scope / very high bar** |

Parser and renderer results are intentionally independent. Track B shows that bounded parsing is attainable; it does not rescue a failed rendering surface. Any several-MiB policy must remain plain-text fallback even with a future renderer.

## 20. Recommended architecture

**Overall architecture decision: do not proceed to Phase 1 on the current UWP editor.** There is no qualified renderer. Keep Notepads plain text and do not add dormant product abstractions, settings, dependencies, or parser startup work.

ColorCode classification is **VIABLE ONLY FOR SELECTED LANGUAGES**. Stock ColorCode is not safe for arbitrary source. If rendering becomes possible after a future platform/control change, the initial parser recommendation is a neutral, buffered adapter using audited ColorCode grammars, a timeout-capable compiler, generation rejection, a serialized bounded worker, and plain-text fallback on timeout. Unsafe/high-value grammars may instead use small stateful span lexers behind the same interface. The 43x dense-JSON prototype advantage makes a span-only custom path worth reconsidering only when the actual v1 language set is deliberately narrowed.

No renderer is recommended for production today. A future editor surface must expose supported, presentation-only text decorations that share the editor's own glyph layout and stay outside undo/history. That is more likely to come from a platform/editor migration than another UWP overlay. Replacing `RichEditBox` solely for this feature remains unjustified.

The original engineering plan is intentionally unchanged. Phase 0B found no sufficiently strong replacement architecture to revise future phases; its old RichEdit range-renderer direction remains superseded by Phase 0's recorded NO-GO, and this report adds the new renderer blocker without rewriting that history.

## 21. Unresolved blockers

- **Blocking:** identify a supported presentation-only renderer that shares the editable control's layout and does not enter undo/history. Native TOM and the whole-document XAML overlay both failed.
- If platform migration is considered, qualify its startup, ordinary-text memory, IME, accessibility, touch, bidi, encoding/save, session, diff/preview, and upstream-rebase impact before treating syntax coloring as justification.
- Before any future ColorCode adoption, audit/fuzz each advertised grammar, especially strings/comments and nested languages; reproduce timeout behavior on every shipped architecture.
- Decide the deliberately narrow language set before choosing between hardened ColorCode and stateful lexers. Missing YAML, Bash, TOML, INI, JSONC, and distinct C support remains as recorded in Phase 0.
- Human IME validation remains mandatory for any future renderer, but it is not useful against the rejected overlay/native candidates.

## 22. Explicit go/no-go gates

| Gate | Result | Consequence |
| --- | --- | --- |
| Native ABI acquisition | **FAIL** — classic TOM QIs return `E_NOINTERFACE` in Debug and Production | `tomApplyTmp`, undo suspension, Freeze/Unfreeze, and native benchmark are unavailable |
| Native UWP supportability | **FAIL** — documented desktop-only and absent at runtime | Native TOM renderer is `INACCESSIBLE` |
| Overlay exact layout | **FAIL** — no display-control equivalent for Notepads' `DefaultTabStop`; other parity unproved | Whole-document overlay is no-go |
| Overlay interactive performance | **FAIL to qualify** — 100 KiB synchronous layout ~82.55 ms; forced raster ~1.28 s and replacement ~2.99 s | Deferred on-screen cost/scrolling remain unproved; 500 KiB synchronous layout is already ~1.24 s |
| Packaged regex timeout | **PASS x64** — custom compiler/API throws and process recovers under Production .NET Native | Bounded ColorCode failures are technically possible |
| Known JSON/JS regex hardening | **PARTIAL PASS** — normal/single-start pathologies fixed; repeated JS candidates remain bounded only by timeout | Stock grammars cannot ship unmodified |
| Span-only feasibility | **PASS as a limited prototype** — dense JSON ~43x faster/~96% less allocation at 1 MiB | Supports future custom-lexer option, not broad language qualification |
| End-to-end safe architecture | **FAIL** | **NO-GO; do not begin Phase 1** |

The hard future gates remain undo/plain-text/event integrity, native IME/input, accessibility, lifecycle, and plain-text fallback on whatever new rendering surface is proposed. Compilation alone never satisfies them.

## 23. Exact reproduction commands

Baseline and source inspection:

```bash
git branch --show-current
git rev-parse HEAD
git status --short
git remote -v
rg -n 'DefaultTabStop|TabStop' '/c/Program Files (x86)/Windows Kits/10/Include/10.0.26100.0/winrt/windows.ui.text.idl' '/c/Program Files (x86)/Windows Kits/10/Include/10.0.26100.0/winrt/windows.ui.xaml.controls.idl'
rg -n 'ITextDocument|tomApplyTmp|tomFalse|Freeze|Unfreeze' '/c/Program Files (x86)/Windows Kits/10/Include/10.0.26100.0/um/TOM.h'
```

Baseline/final solution builds (run separately for each configuration):

```bash
MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' src/Notepads.sln /p:Platform=x64 /p:Configuration=Debug /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /m /nologo /v:minimal
MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' src/Notepads.sln /p:Platform=x64 /p:Configuration=Production /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /m /nologo /v:minimal
```

The temporary runtime harness was conditionally launched from `App.OnLaunched`, wrote JSON to `ApplicationData.Current.LocalFolder`, and was packaged after a solution build with:

```bash
MSYS2_ARG_CONV_EXCL='*' '/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe' src/Notepads/Notepads.csproj /p:Platform=x64 /p:Configuration=Production /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never /p:GenerateAppxPackageOnBuild=true /p:BuildProjectReferences=false /m /nologo /v:minimal
```

The uniquely named unsigned research MSIX was installed and launched with `Add-AppxPackage -AllowUnsigned` and `shell:AppsFolder\<package-family>!App`. Its saved Debug and Production JSON was copied outside package state before uninstall. The harness queried IIDs with balanced `Marshal.Release`, exercised `TextHighlighter` ranges/forced rendering, and passed ColorCode a custom timeout compiler. Every hook, project define/reference, manifest change, package, corpus, and out-of-tree source clone was removed before final builds.

Track B's disposable source benchmark can be recreated from the authoritative tag; the original temporary clone/harness was deleted during cleanup:

```powershell
git clone --depth 1 --branch v2.0.15 https://github.com/CommunityToolkit/ColorCode-Universal.git E:\Notepads-phase0b-colorcode
git -C E:\Notepads-phase0b-colorcode rev-parse HEAD
# Expected: e6c2701c365a7a91d74d7ef38a6b60075008ce94
```

Recreate the isolated rules and span scanner described in sections 14–17 in a disposable Release console harness, run each case in a separate process, and give every regex a finite timeout. Do not run the stock pathological expressions with an infinite timeout in the parent process. The benchmark harness itself is intentionally not retained as repository code because this phase requires a documentation-only final diff.
