# Repository Guidelines

## Project Structure

The solution is `src/Notepads.sln`. `src/Notepads/` is the packaged UWP app: pages are under `Views/`, editor and dialog UI under `Controls/`, tab/session coordination under `Core/`, application-wide state under `Services/`, and file/encoding helpers under `Utilities/`. Assets and localized `.resw` files live in `Assets/` and `Strings/<language-code>/`. `src/Notepads.Controls/` contains reusable XAML controls and the vendored Markdown renderer. CI configuration is in `.github/workflows/` and `azure-pipelines.yml`; documentation screenshots belong in `ScreenShots/`.

## Architectural Boundaries

`NotepadsCore` owns tab and `ITextEditor` lifecycles. `TextEditor` owns file identity, saved snapshots, encoding/line-ending choices, preview/diff modes, and save/reload state. `TextEditorCore` subclasses `RichEditBox` and owns document ranges, input, undo/redo, selection, scrolling, and editor rendering. Keep disk access in `FileSystemUtility`; do not add it to controls. The `Extensions` provider is currently a hard-coded Markdown content-preview seam, not a general editor plugin API. Preserve these boundaries and minimize divergence from upstream.

## Build and Run

Use Windows with Visual Studio 2022 (or compatible MSBuild), UWP support, and Windows SDK 10.0.22621.0. From a Visual Studio Developer Command Prompt:

- `msbuild src\Notepads.sln /t:Restore /p:Platform=x64 /p:Configuration=Debug`
- `msbuild src\Notepads.sln /p:Platform=x64 /p:Configuration=Debug /p:AppxPackageSigningEnabled=false /p:AppxBundle=Never`

Open the solution, select `Debug | x64`, and press `F5` to deploy interactively. Do not substitute `dotnet build`; these are legacy UWP projects. In Git Bash, prefix MSBuild with `MSYS2_ARG_CONV_EXCL='*'` so `/p:` arguments are not rewritten. CI additionally packages Debug, Release, and Production for x86, x64, and ARM64. Never commit `bin/`, `obj/`, `AppPackages/`, or generated MSIX artifacts.

## Style and Naming

Follow `src/.editorconfig`: four spaces, Allman braces, and CRLF for C#, XAML, project, and solution files. Use PascalCase for types/public members, `I`-prefixed interfaces, `_camelCase` private fields, camelCase parameters, and `Async` suffixes. Preserve paired XAML/code-behind names and focused partial files such as `TextEditorCore.LineNumbers.cs`.

## Validation

There is no automated test project or coverage threshold. Build the full solution, then manually exercise the affected packaged-app workflow. Editor changes require checks for open/save/reload, encoding and line endings, selection, undo/redo, paste and IME/RTL input, large-file responsiveness, light/dark/high-contrast themes, and session restoration. UI changes need relevant window-size and localization checks. Report baseline warnings separately from new failures.

## Commits and Pull Requests

Inspect `git status` and preserve unrelated changes. Use concise Conventional Commit-style titles: `fix:`, `feat:`, `doc:`, `ci:`, `lang:`, or `other:`. Complete `.github/PULL_REQUEST_TEMPLATE.md`, link the issue, describe validation, and add before/after screenshots for visible changes.
