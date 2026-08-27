# React Native Master

> An enterprise-grade React Native / TypeScript engineering standard, packaged as a [Claude Code](https://claude.com/claude-code) skill.

![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-8A63D2)
![React Native](https://img.shields.io/badge/React%20Native%20%2F%20TypeScript-61DAFB?logo=react&logoColor=black)
![License: MIT](https://img.shields.io/badge/license-MIT-green)

**React Native Master** makes an AI assistant behave like a Staff/Principal React Native Engineer performing a code review *before* writing any code. It is not a feature factory — it turns code generation into **code engineering**: decide the right shape first, then produce production-grade code suitable for large-scale enterprise apps.

Once installed, it applies automatically whenever you write, refactor, review, architect, test, secure, or optimize React Native / TypeScript code — you never have to say "use clean architecture" again.

## Quick start

```bash
git clone <your-fork-url> react-native-skills
cd react-native-skills
./install.sh
```

Restart Claude Code. That's it — the skill triggers on any React Native / TypeScript task.

`install.sh` symlinks `~/.claude/skills/react-native-master` to this repo, so editing a reference file here is live immediately. See [Install](#install) for the other modes.

## What actually changes

Ask for "a profile card component" and a generic assistant gives you this:

```tsx
const renderProfileCard = (user) => (            // render function, not a component
  <View style={{                                 // inline style object, new every render
    padding: 16,                                 // magic number
    backgroundColor: '#F5F5F5',                  // raw hex, breaks dark mode
    borderRadius: 12,
    flexDirection: 'row',
  }}>
    <Image source={{ uri: user.avatar }} />      {/* unsized, layout jumps on load */}
    <Text style={{ fontSize: 16, fontWeight: '600' }}>
      {user.fullName}                            {/* pushes siblings off screen */}
    </Text>
  </View>
);
```

With this skill, the same request produces an exported component in its own file, values from the project's theme tokens, a `StyleSheet`, a truncation policy on API-driven text, a sized and cached image, a real press target with an accessibility role — and a loading/empty/error state for the screen it lives on, or a question when a token is genuinely missing, instead of an invented hex.

The rules behind each of those decisions live in the reference files below.

## The Golden Rule

> **Never write code just because it works.**
> Always write production-grade code. Prioritize readability over cleverness. Every architectural decision must be justified by scalability, maintainability, and testability. Minimize coupling, maximize cohesion, follow SOLID, and align with official React, React Native, and TypeScript guidelines.

If a request violates this rule, the assistant says so plainly and proposes the correct shape. Being a good engineer sometimes means pushing back.

## How it works

1. A task touches a `.tsx`/`.ts` file or a React Native concept → the standard triggers (even if you never say "architecture" or "review").
2. The task is named as one of three **build modes** — *scaffold* (new feature), *UI* (draw a screen), or *composition/refactor* (clean up existing code).
3. The orchestrator loads only the 2–3 relevant reference files. A new feature is usually `scaffold` + `patterns` + `testing`; drawing a screen is `ui-from-design` + `ui-layout` + `refactor` §8; a review is `review` plus the domain the code lives in.
4. The design is decided first, then production-grade code is written.
5. A self-review checklist runs at the end of every code response, and anything with runtime behavior gets a manual self-test **on both platforms**. Rule violations are flagged, never applied silently.

This is **progressive disclosure**: `SKILL.md` is a small orchestrator, and the deep rules stay out of context until the task needs them.

## What's inside

| File | Covers |
| --- | --- |
| [`SKILL.md`](SKILL.md) | Orchestrator — Golden Rule, pre-code checklist, default stack, three build modes, routing table, self-review protocol |
| [`ui-from-design.md`](references/ui-from-design.md) | Drawing UI from a screenshot/Figma — decompose → measure → ask → build → verify, the token contract, shape recipes, reusable-component inventory, design-comparison loop |
| [`ui-layout.md`](references/ui-layout.md) | Flexbox vs CSS, the layout failure table, canonical screen shapes, safe area + keyboard, responsive & font scale, platform differences, the four screen states, accessibility |
| [`scaffold.md`](references/scaffold.md) | Creating a feature end to end — repo calibration, dependency rules, domain-first build order, layer templates, typed navigation, directory layout |
| [`architect.md`](references/architect.md) | Clean Architecture, the dependency rule, feature-first folders, SOLID, DDD with branded types, lint-enforced boundaries |
| [`patterns.md`](references/patterns.md) | `Result` type, Repository, Mapper (DTO⇄Entity), zod at the boundary, UseCase, DI composition root, server vs client state, Zustand shape |
| [`performance.md`](references/performance.md) | Re-render scoping, list virtualization (FlashList/FlatList), cleanup & leaks, Reanimated, images, startup, profiling |
| [`security.md`](references/security.md) | Secrets in a readable JS bundle, Keychain/Keystore, single-flight 401 refresh, HTTPS + pinning, OAuth/PKCE, deep links, WebView, OTA hazards |
| [`testing.md`](references/testing.md) | Test pyramid, Jest + RNTL, MSW, fakes via the DI container, snapshot discipline, Maestro/Detox E2E, >90% business-logic coverage, TDD |
| [`refactor.md`](references/refactor.md) | KISS/DRY/YAGNI, component extraction, god components & hooks, magic values, smell→fix table, the component-composition rulebook (§8) |
| [`review.md`](references/review.md) | PR review order, full checklist, smell detectors, AI self-review protocol |
| [`best-practices.md`](references/best-practices.md) | Strict TypeScript, ESLint rules that catch defects, naming, discriminated unions, hooks discipline, immutability, prefer/avoid lists |
| [`manual-test.md`](references/manual-test.md) | Proving a change at runtime — static gates, test plan, launch & drive, evidence capture, the Honesty Rule, both platforms |
| [`enterprise.md`](references/enterprise.md) | CI/CD gates, flavors & EAS profiles, OTA discipline, monorepo, ADRs, semver + source maps, observability, native upgrade hygiene |

## Default tech stack

Unless your project dictates otherwise:

- **Language** — TypeScript `strict`. No `any`, no silencing assertions.
- **Architecture** — Clean Architecture, feature-first. `presentation → domain ← data`. The domain layer depends on nothing.
- **Server state** — TanStack Query owns fetching, caching, retries, invalidation.
- **Client state** — Zustand (or RTK) with selector-scoped reads; local UI state stays local.
- **Models** — zod validates every DTO at the boundary; entities are plain immutable types with branded ids.
- **Errors** — `Result<Failure, T>` discriminated union from the domain boundary. No exceptions crossing layers.
- **DI** — Composition root + typed context. No module singletons reached into from components.
- **Networking** — One configured client with interceptors; a mapper between DTOs and entities.
- **Navigation** — React Navigation / Expo Router with a typed param list.
- **Styling** — Theme tokens + `StyleSheet.create`. No raw hex, no inline style objects.
- **Lists** — FlashList / FlatList with memoized rows and stable keys.

**Prefer:** function components · `Pressable` · typed props · derived state · `memo` where it pays · selector-scoped store reads · `Result` types · zod at the boundary · Repository & UseCase patterns · feature-first structure · safe-area insets from the hook.

**Avoid:** god components · business logic in JSX · magic numbers/hex · deep prop drilling · `useEffect` for derivation or fetching · `any` · `TouchableWithoutFeedback` on real controls · `ScrollView` for long lists · inline styles in list items · global mutable singletons · module-scope `Dimensions.get()`.

## Adapting it to your project

**It works on a fresh clone.** Two reference files open by calibrating to whatever repo they're pointed at, before writing a line:

- [`scaffold.md`](references/scaffold.md) §0 — reads your reference feature to learn your `Result` implementation, file naming, DTO validation, state libraries, DI style, and HTTP client.
- [`ui-from-design.md`](references/ui-from-design.md) *Calibrate* — finds your theme tokens, styling approach, icon/image components, i18n accessor, tap primitive, and shared-component folder.

The concrete names throughout those files (`theme.spacing.md`, `AppPressable`, `src/core/ui/`) are a **generic worked example**, clearly marked as such. The rules around them are universal.

One optional tune-up if you're forking for a team:

- [`ui-from-design.md`](references/ui-from-design.md) §1 + §4 — replace the example token table and shared-component inventory with your project's real ones. Calibration discovers them each session; a written inventory means the assistant reuses your components without having to go looking.

Everything else — architecture, patterns, performance, security, testing, review, layout, TypeScript idioms, manual testing — applies to any React Native codebase as written, Expo or bare.

## Repository layout

The repo **is** the skill — same layout Claude Code expects, so there is no second copy to keep in sync.

```
SKILL.md            # orchestrator (skill entry point)
references/*.md     # the deep rules, loaded on demand
install.sh          # symlink this repo into ~/.claude/skills/
build.sh            # package react-native-master.skill (gitignored build artifact)
```

## Install

```bash
./install.sh              # symlink ~/.claude/skills/react-native-master -> this repo (recommended)
./install.sh --copy       # copy instead of symlink (re-run after every edit)
./install.sh --uninstall  # remove the link
./build.sh                # produce react-native-master.skill for distribution
```

The symlink install means editing a reference here is live immediately — no sync step, no drift between the repo and the installed copy. An existing real directory is moved to `~/.claude/skill-backups/` first, never inside `~/.claude/skills/` (a leftover copy there loads as a duplicate skill).

Restart Claude Code, or start a new session, to pick up the change.

## Using it without Claude Code

The reference files are plain markdown and work as a standalone React Native engineering handbook, a team onboarding doc, or a PR-review checklist. Paste a single file into any assistant that accepts long context, or hand `references/review.md` to a human reviewer.

## Contributing

Issues and PRs welcome. Two rules for reference edits:

1. **Rules, not essays.** Every addition should be actionable — a rule, a table, a checklist item, or a copy-pasteable recipe. If a paragraph doesn't change what gets written, cut it.
2. **Keep the routing honest.** A new reference file needs a row in the `SKILL.md` routing table and in the table above, or it will never be loaded.

## License

[MIT](LICENSE)
# react-native-skilss
