---
name: appmetrica-yandex-versions
description: Finds which AppMetrica Unity plugin version goes with which AppLovin MAX Yandex Ads adapter version, in either direction, and reports the nearest workable alternatives when no exact pair exists. Use this whenever someone names a version of either one and asks what to pair with it, plans a bump of either plugin, or mentions `io.appmetrica.analytics`, `com.applovin.mediation:yandex-adapter`, or `com.yandex.android:mobileads` in a compatibility context — including loose phrasings like "can I upgrade AppMetrica to X", "which yandex adapter for Y", "will these two versions work together". The two SDKs share a native dependency and a mismatch resolves silently, so always check the pairing rather than answering from memory.
---

# AppMetrica ↔ Yandex Ads version matching

## The problem this solves

The AppMetrica Unity plugin and the AppLovin MAX Yandex adapter both pull in the **same native AppMetrica SDK**. When their versions drift, Gradle resolves the skew without a warning — the build is green and the SDK combination is one nobody tested. There is no error message to work backwards from, which is why picking the version deliberately is worth the lookup.

## The three version spaces

Users hand you a version from any of these. Work out which one first, then translate.

1. **AppLovin MAX Yandex adapter** — `com.applovin.mediation:yandex-adapter:A.B.C.D` (e.g. `7.16.0.0`). Lives in `Assets/MaxSdk/Mediation/Yandex/Editor/Dependencies.xml`.
2. **Yandex Mobile Ads SDK** — `A.B.C` (e.g. `7.16.0`). Never named directly in a Unity project, but it is the hinge everything else is expressed in.
3. **AppMetrica Unity plugin** — `6.x.y` (e.g. `6.7.0`), from `Packages/manifest.json`. Wraps a **native** AppMetrica Android SDK — that native number is what actually has to line up.

Link rules:

- **Adapter → Yandex SDK**: drop the fourth segment. `7.16.0.0` → Yandex SDK `7.16.0`. The fourth segment is AppLovin's own patch counter and never affects AppMetrica compatibility.
- **Yandex SDK → native AppMetrica**: look it up in the matrix.

## Workflow

**Step 1 — Identify the input.** Four segments means an adapter; `6.x` means the Unity plugin; a bare three-segment number is ambiguous, so infer from context or ask.

**Step 2 — Read `references/version-matrix.md`.** It has the lookup tables and a pre-computed list of matching pairs. Read it every time — these numbers follow no pattern you can reason your way to, so memory and inference are both unreliable here.

**Step 3 — Match on Android.** Yandex's Gradle module metadata declares two things about `io.appmetrica.analytics:analytics`: `prefers` (the version Yandex actually tested against — identical to the changelog's "recommended AppMetrica version") and `strictly` (a hard range). **Match `prefers` exactly.** A satisfied `strictly` range is not a match; it only means Gradle will not complain.

The plugin declares its native SDK as a plain version, which Gradle reads as *at least* that. So the three states behave differently, and it is worth naming which one a user is in:

- **Plugin above `prefers`, inside `strictly`** — resolves silently at the plugin's version. Untested combination, no signal.
- **Plugin below the `strictly` floor** — also silent, but Gradle *raises* the native SDK to satisfy Yandex, so the plugin's C# wrapper drives a native it was not built against.
- **Plugin at or above the `strictly` ceiling** — nothing satisfies both and the build fails. The only skew that announces itself.

**Step 4 — Check iOS only when it changes the answer.** Most pairings are fine on iOS and mentioning it just adds noise. It matters from Yandex Ads **7.16.0 onward**, where the CocoaPods constraint on `AppMetricaCore` narrows to `~> A.B.C` — a single minor. Below 7.16.0 the constraint is wide (`>= X, < 6.0.0`) and effectively always satisfied.

Since AppMetrica Unity pins its iOS pods to an exact version (up to 6.9.0; 6.10.0 uses a range), a pair can match perfectly on Android and be impossible on iOS. The matrix flags these. Raise it when the recommended pair is one of them — a failed `pod install` is a cheap thing to warn about and an annoying thing to discover. Otherwise stay quiet about iOS.

**Step 5 — If there is no exact pair, find the nearest.** This is the common case and the most useful thing you produce. Search outward in both directions:

- **Down** — the newest older pairing that matches. Usually the boring, safe answer.
- **Up** — the oldest newer pairing that matches, with its cost named: Yandex Ads 8.x is a major API rewrite, and AppMetrica Unity 6.8.0 removed `reportReferralUrl` and the `AndroidPlayer` module.

Say which direction each option lies in and how far — "откатить адаптер на два минора, до 7.14.1.0" is actionable where a bare version number is not. When the user has pinned one side, answer for the other rather than surveying everything. And if nothing lines up, say so plainly and name the best compromise: a mismatch someone chose knowingly is fine, one they walked into is not.

## Output format

Answer in Russian, unless the user clearly writes in another language — then mirror theirs. Translate the labels; keep version numbers and file paths as-is.

```
**Вердикт:** ✅ точное совпадение / ⚠️ только ближайшее / ❌ ничего не сходится

| Yandex adapter | Yandex Ads SDK | нативная AppMetrica | AppMetrica Unity |
|---|---|---|---|
| 7.16.0.0 | 7.16.0 | 7.12.0 (prefers) | 6.7.0 → 7.12.0 ✅ |

[если не точное] **Ближайшие варианты**
- ↓ вниз: <версия> — <что менять, насколько сдвиг>
- ↑ вверх: <версия> — <что менять, чем платим>

[только если пара ломает iOS] ⚠️ iOS: <ограничение> против пина <версия> — pod install не разрешится
```

Keep it to that. The user wants a version number and a file to edit.

Where the change lands:
- adapter → `Assets/MaxSdk/Mediation/Yandex/Editor/Dependencies.xml`, **both** the `androidPackage` and the `iosPod` line, then Force Resolve in EDM4U
- AppMetrica Unity → `Packages/manifest.json`, the `io.appmetrica.analytics` git tag

## When the version is not in the tables

The matrix is a snapshot and both SDKs ship every few weeks. If the version you were asked about is missing — whether newer than the snapshot or a gap in the middle — **refresh before answering**. `references/version-matrix.md` ends with a script that regenerates the whole table from Maven module metadata and CocoaPods specs in under a minute.

Refreshing is cheap; a wrong pairing is not, because nothing surfaces it until much later. When you refresh, write the new rows back into the matrix so the next run starts from better data.

**A missing version may also simply not exist.** People relay versions secondhand and get them wrong, or name one that was announced but not published. Before pairing anything against a version you cannot find, confirm it is real — the matrix's refresh section has the existence checks. Saying "there is no such version, the newest is X" is a useful answer; inventing a pairing for a version nobody can install is not.
