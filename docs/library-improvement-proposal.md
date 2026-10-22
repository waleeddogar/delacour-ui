# Library improvement proposal

Review date: 2026-10-22. Source: `develop` at `36b3b80f`.

This proposal ranks five improvements to the library and its delivery tools.
It records source findings, implementation scope, and acceptance checks.
It does not change component behavior or declare the library ready for release.

The library already has broad component coverage. It also has compound APIs,
theme tokens, direction support, reduced motion, and shared overlay behavior.
The CLI supports updates and copies tests for Bun, Jest, and Vitest.
The playground has interaction flows and visual comparison tools.
Keep these systems. Extend their coverage where consumers can still encounter gaps.

| Rank | Improvement | Consumer benefit | Relative effort |
| --- | --- | --- | --- |
| 1 | Verify the UI package as an npm tarball | Catch failures in the second installation path before release | Medium |
| 2 | Add focused native interaction checks to signoff | Catch failures that types, pure tests, and images cannot detect | Large |
| 3 | Make calendar messages configurable | Let applications use one language for dates and spoken instructions | Medium |
| 4 | Publish and check the supported native configuration | Report unsupported setups before a native build fails | Medium |
| 5 | Make duplicate-module diagnostics precise | Avoid false warnings in linked workspaces | Small |

Rank reflects consumer impact. Effort is a relative estimate, not a delivery date.
Items 3 and 5 have specific source findings. Items 1, 2, and 4 address coverage gaps.
No runtime failure was reproduced during this review.

## 1. Verify the UI package as an npm tarball

**Evidence.** The [verification harness](../packages/cli/scripts/verify/harness.ts)
packs the chart and bottom sheet engines in `WORKSPACE_PACKAGES`.
It installs the UI components through the CLI copy path.
The [UI package rules](../packages/react-native-ui/AGENTS.md#publishing)
identify npm installation as a separate path that needs a tarball check.
The current engine checks must stay.

**Proposed change.** Extend the existing Expo verification harness with a package
installation mode. Pack `@delacour/react-native-ui` from the candidate commit.
Install it in an isolated Expo app. Import components through public subpaths.
Use the current engine tarballs when a selected component needs them.
Keep the source as TSX. Keep the current per-component exports.

**Scope.** Update `packages/cli/scripts/verify-expo.ts`, its `verify/` helpers and
fixtures, and the relevant release checks in `.github/workflows/ci.yml`.
Check `packages/react-native-ui/package.json` exports and shipped files.
Update the installation guide and package verification instructions.
Run this check for release candidates first to limit installation cost.

**Acceptance checks.**

- A minimal consumer bundles `Button` without installing optional chart, sheet, or map peers.
- A second consumer bundles `Chart` and `BottomSheet` with their engine tarballs.
- TypeScript and Metro resolve public subpaths and the style entry from the tarball.
- Uniwind generates the classes used by the sample components.
- The consumer resolves no source from the repository checkout.
- The check removes its temporary app and install cache after completion or failure.

**Risk.** Installing all optional peers in every fixture would hide dependency leaks.
Keep the minimal consumer separate from the engine consumer.

## 2. Add focused native interaction checks to signoff

**Evidence.** The [test rules](../packages/react-native-ui/AGENTS.md#testing)
explain that most library tests exercise pure logic.
The [preview instructions](../apps/playground/AGENTS.md#capturing-preview-media)
already provide Warden flows, image comparisons, and clip checks.
They also document a timing limitation that prevents correct chart scrub captures.
The [signoff checks](../scripts/signoff-checks.ts) do not run native interactions.
React Native explains why JavaScript component tests cannot verify native code.
See its [testing guidance](https://reactnative.dev/docs/next/testing-overview).

**Proposed change.** Reuse the playground and Warden flows for a small set of
behavior assertions. Start with `Dialog`, `BottomSheet`, `Input`, and `Slider`.
Check outcomes as well as images. Add a local signoff check for changes to these
components or their shared code. Use the existing CI dispatch path when a local
runner is unavailable. Keep ordinary PR checks local.

**Scope.** Update playground flows, small demos where state needs to be visible,
`scripts/signoff-checks.ts`, its tests, and the signoff report.
Map changes in shared press, overlay, keyboard, and direction code to the selected flows.
Add an Android run after checking whether the existing runner can support it.
Record that capability check before selecting any new tool.

**Acceptance checks.**

- A dialog opens, blocks background input, closes, and returns focus to its trigger.
- A sheet keeps an input visible when the keyboard opens and closes.
- Android Back dismisses the top overlay without dismissing the screen below it.
- A controlled slider returns to the supplied value when its parent rejects a change.
- RTL and reduced motion retain the same interaction results.
- VoiceOver and TalkBack checks record labels, focus order, and focus return.
- A report distinguishes automated assertions from manual screen reader checks.
- A missing runner or failed capture cannot produce a passing signoff result.

**Risk.** Native runs need a compatible binary, available disk space, and stable timing.
Start with the existing iOS runner. Do not claim Android coverage before its run passes.
Keep chart scrub assertions blocked until the documented gesture timing limit is resolved.
Record native behavior separately from image comparison results.
Only automated assertions determine this signoff result. Record manual screen
reader evidence separately. A passing automated check does not imply accessibility approval.

## 3. Make calendar messages configurable

**Evidence.** [Calendar day hints](../packages/react-native-ui/src/components/calendar/calendar-day.tsx)
use fixed English strings for today and range boundaries.
[Calendar caption actions](../packages/react-native-ui/src/components/calendar/calendar-caption.tsx)
also use fixed English instructions.
[Date picker value and announcement helpers](../packages/react-native-ui/src/components/date-picker/date-picker.variants.ts)
format dates with `locale`, but add English phrases such as `not set` and `Selected`.
The day props do not provide a direct hint override.
Thus, a locale can change the date format while leaving the instruction in English.
These labels and hints form part of the spoken interface, as described in the
[React Native accessibility guide](https://reactnative.dev/docs/accessibility).

**Proposed change.** Add optional, typed message overrides to `Calendar` and
`DatePicker`. Include functions where a message needs dates or a count.
Pass overrides through the existing contexts and date picker calendar adapter.
Keep English defaults. Preserve current `doneLabel`, `clearLabel`, and formatter props.
Let applications supply translations without adding a translation dependency.

**Scope.** Update calendar and date picker props, contexts, parts, pure message
helpers, tests, public type exports, and component instructions.
Add one non-English demo and one RTL demo using the existing `Direction` component.
Update the web guides, registry metadata, previews, and a UI package changeset.
Inspect composed date controls before choosing the final override names.

**Acceptance checks.**

- Existing callers produce the same English text without new props.
- A supplied message set controls day hints, navigation actions, empty values, and selection announcements.
- Range and multiple-selection messages use the application's supplied formatters.
- A non-English demo has no English fallback in the messages that it overrides.
- Invalid locales retain the current safe date-format fallback.
- VoiceOver and TalkBack read the supplied messages in the demo.

**Risk.** Fragment concatenation does not fit every language.
Use message functions for complete sentences. Do not ship unreviewed translations.

## 4. Publish and check the supported native configuration

**Evidence.** The [verification app](../packages/cli/scripts/verify/harness.ts)
pins one Expo, React, and React Native configuration.
The [UI peer ranges](../packages/react-native-ui/package.json) accept broader
combinations, including open-ended ranges for Reanimated and Worklets.
Those ranges alone do not show which combinations have passed verification.
The [installation guide](../apps/web/content/docs/native/getting-started/installation/index.mdx)
documents the Skia 3 Android requirement.
The [playground config](../apps/playground/app.config.ts) sets Android API 26.
The [doctor checks](../packages/cli/src/commands/doctor.ts) check New Architecture,
but do not check that Skia-specific Android setting.

**Proposed change.** Document one verified native configuration first.
Separate verified support from versions allowed by peer ranges.
Extend `doctor` to report the Skia Android requirement when the app uses Skia.
Check resolved Expo configuration where possible. Report unknown configuration
as unknown rather than passing it. Name the native rebuild step in the fix.

**Scope.** Update doctor checks and fixture tests, the installation guide, and
the engine guides. Compare the manifests with the verification harness and root
catalog. Reuse existing version data where it is authoritative.
Assess any peer-range change separately because it can reject existing installs.

**Acceptance checks.**

- The supported configuration names exact versions and the platforms actually checked.
- A verified configuration passes the relevant doctor checks.
- A Skia app with a known Android minimum below 26 receives an actionable diagnostic.
- An app without Skia receives no Skia-specific warning.
- Missing or unreadable native configuration receives an explicit unknown result.
- The documentation distinguishes Metro verification from a successful native build.

**Risk.** A JavaScript config cannot prove device GPU support or the installed binary's contents.
Keep those limits explicit. Do not promise a full compatibility matrix from one successful build.

## 5. Make duplicate-module diagnostics precise

**Evidence.** `checkDuplicateNativeModules` in
[doctor.ts](../packages/cli/src/commands/doctor.ts) compares path existence in
the app and workspace `node_modules` directories.
It does not compare real paths. Two links to one physical package can therefore
produce the same warning as two separate copies.
This is a source finding, not a reproduction against a consumer workspace.

**Proposed change.** Resolve the candidate package directories to real paths.
Warn when both paths resolve and differ. Keep the warning for actual duplicate
copies. Report filesystem errors without silently claiming that the setup is valid.

**Scope.** Update the existing doctor helper and
`packages/cli/src/commands/doctor.test.ts`. Add small temporary workspace fixtures.
Preserve the JSON result shape used by the CLI and MCP command.
Update CLI instructions and add a CLI changeset.

**Acceptance checks.**

- Two symlinks to one package directory do not trigger the duplicate warning.
- Two separate package directories still trigger the warning, even at the same version.
- A normal single-root installation keeps its current result.
- A broken link or unreadable path produces an explicit diagnostic.
- Fixture cleanup removes only files created by the test.

**Risk.** Different physical paths do not prove that Metro loads both copies.
Keep this a warning. Do not rewrite Metro configuration automatically.

## Suggested delivery order

Start with item 5 as a small correction. Implement item 1 before the next release.
Then implement item 3 as a bounded API enhancement.
Use item 4 to define the native configurations for item 2.
Deliver each item in its own PR with the checks listed above.

Defer new component families, a new renderer, a replacement test framework,
and a broad performance rewrite. This review found no measured need for them.
The proposal requires no runtime changes, generated registry changes, or changeset.

## Review validation and limits

The review sampled the UI package, CLI, engine verification, playground flows,
and release checks. It did not inspect every component or perform a security audit.

- `bun run check` passed with existing warnings.
- The focused doctor and date picker suites passed: 80 tests, zero failures.
- `bun run typecheck` failed because the installed tree lacks `react-native-skia`.
- `bun run test` failed with missing Skia and `@delacournz/otel-start` imports.
- No dependency install ran. The disk had about 1.2 GiB free, below the repository limit.
- The required cleanup helper could not reach Docker and reclaimed no space.
- No build, simulator run, screen reader check, or full signoff ran.

These results describe this checkout. They do not establish failures in a fresh,
correctly installed consumer app. Keep the proposal PR in draft until the required
signoff passes on a machine with the correct dependencies and enough free space.
