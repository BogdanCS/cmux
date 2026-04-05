 Contributor Expectations

  Prerequisites

   - macOS 14+, Xcode 15+, Zig (brew install zig)
   - Clone with --recursive (submodules required)
   - Run ./scripts/setup.sh once

  PR Format

  Every PR uses this template:

   ## Summary
   - What changed?
   - Why?
   
   ## Testing
   - How did you test this change?
   - What did you verify manually?
   
   ## Demo Video
   For UI/behavior changes, include a short demo video.
   
   ## Review Trigger (Copy/Paste as PR comment)
   @codex review
   @coderabbitai review
   @greptile-apps review
   @cubic-dev-ai review
   
   ## Checklist
   - [ ] I tested the change locally
   - [ ] I added or updated tests

  Key Code Policies

   - Localization: All user-facing strings must use String(localized:) — no bare string literals in UI
   - Shortcuts: New keyboard shortcuts must go in KeyboardShortcutSettings + config docs
   - Tests: Must verify observable runtime behavior — no grep-style source-inspection tests
   - Regression tests: Use a two-commit structure (failing test first, then fix) so CI proves coverage
   - Never run tests locally — trigger via GitHub Actions
   - Socket commands: Default to off-main thread; main-thread use requires explicit justification
   - Socket focus: Commands must not steal app focus unless they're explicit focus-intent commands
   - Submodules: Push submodule commits to remote main before updating the parent repo pointer

  Build Convention

  Always use ./scripts/reload.sh --tag <your-branch-slug> — never bare xcodebuild or untagged debug
  builds.

  License

  Contributions are GPL-3.0-or-later, with a CLA granting Manaflow Inc. a commercial sublicense right.