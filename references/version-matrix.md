# Version matrix

Snapshot **2026-10-09**. Newest known: Yandex Ads 8.6.0 (Android + iOS), AppLovin Yandex adapter 8.6.0.0, AppMetrica Unity 6.10.0.

Re-verified 2026-10-09 against Maven, CocoaPods, the AppLovin integration manager and the plugin's git tags. New since 2026-09-02: Yandex Ads 8.5.0 and 8.6.0, `yandex-adapter` 8.5.0.0 and 8.6.0.0 (8.4.0.1, 8.5.0.1, 8.6.0.1 and 8.7.0.0 return 403). AppMetrica Unity is still 6.10.0 — `main` was bumped to 6.11.0 development on 2026-08-03 but still declares native Android 8.4.1, and there is no 6.11.0 tag. Native AppMetrica has moved on without it: Android up to 8.6.0, iOS up to **7.0.0**.

Every number below was derived from machine-readable package metadata, not from prose changelogs — see [Refreshing](#refreshing-this-file). The changelogs agree with it, but the metadata is what the build tools actually obey.

Contents:
- [How the constraints actually work](#how-the-constraints-actually-work) — read this before using the tables
- [Green pairs](#green-pairs)
- [Table A: AppMetrica Unity → native SDKs](#table-a)
- [Table B: Yandex Ads → required AppMetrica](#table-b)
- [Refreshing this file](#refreshing-this-file)

---

## How the constraints actually work

**Android.** `com.yandex.android:mobileads:X` publishes Gradle module metadata declaring, on `io.appmetrica.analytics:analytics`, both a `strictly` range and a `prefers` value:

```
analytics { strictly: "[7.12.0,8.0.0)", prefers: "7.12.0" }
```

`prefers` **is** the "recommended AppMetrica version" from the changelog. Match it exactly.

The plugin declares its native SDK as a plain version, which Gradle reads as *at least* that. So the three skew states behave differently, and only one of them announces itself:

- **Plugin above `prefers`, inside `strictly`** — resolves silently at the plugin's version. Untested combination, no signal at all.
- **Plugin below the `strictly` floor** — also silent, but Gradle *raises* the native SDK to satisfy Yandex, leaving the plugin's C# wrapper driving a native it was not built against.
- **Plugin at or above the `strictly` ceiling** — nothing satisfies both and the build fails. The only skew you get for free.

**iOS.** This is *not* a floor check, despite how the changelog words it. `YandexMobileAds` declares a CocoaPods constraint on `AppMetricaCore` (and friends), and its shape changed over time:

- Yandex ≤ 7.15.x: `>= X, < 6.0.0` — genuinely wide, easy to satisfy.
- Yandex ≥ 7.16.0: `~> A.B.C` — which in CocoaPods means `>= A.B.C, < A.(B+1).0`. **One minor, and that is all.**

Meanwhile AppMetrica Unity ≤ 6.9.0 pins `AppMetricaAnalytics` to an *exact* version, and `AppMetricaAnalytics X` in turn requires `AppMetricaCore = X` exactly. Two exact pins that disagree do not negotiate — `pod install` fails.

**6.10.0 is the turning point**: it declares `AppMetricaAnalytics` / `AppMetricaLibraryAdapter` as `~> 6.3` (i.e. `>= 6.3, < 7.0`), so CocoaPods can settle on whatever minor Yandex asks for inside 6.x. It is the first plugin release designed to coexist with an Ads SDK, and on iOS it is the only one that flexes.

That flex ends at 7.0. AppMetrica iOS 7.0.0 is already on CocoaPods; the first Yandex Ads release that asks for `AppMetricaCore ~> 7.x` will break `pod install` against 6.10.0 the same way the exact pins did. Check for that whenever a Yandex Ads newer than this snapshot comes up.

Consequence worth internalising: **a pair can be perfect on Android and impossible on iOS**, and that combination is common from Yandex 7.16.0 onward. Below 7.16.0 the iOS constraint is wide enough that it never decides anything — so check it, but only bring it up when it actually rules a pairing out.

### iOS: a satisfiable pod pairing can still fail to link (Yandex 8.x)

Version resolution is not the only iOS failure mode from Yandex Ads **8.x** onward, and this one is not a version problem at all — so no bump fixes it.

`YandexMobileAds.xcframework` 8.x is Swift and calls `AnalyticsLibraryAdapter.shared` from `MobileAdsCore.MACAppMetricaAdapterImpl`. `AppMetricaLibraryAdapter` is a **source** Swift pod, so the reference is a raw Swift ABI symbol — `$s24AppMetricaLibraryAdapter09AnalyticscD0C6sharedACvau`, the `unsafeMutableAddressor` of a stored `public static let`. That symbol exists only in a **non-resilient** build. Build the pod with `BUILD_LIBRARY_FOR_DISTRIBUTION = YES` and access goes through a getter instead, the addressor disappears, and the link fails:

```
Undefined symbols for architecture arm64:
  "AppMetricaLibraryAdapter.AnalyticsLibraryAdapter.shared.unsafeMutableAddressor"
      referenced from: MobileAdsCore.MACAppMetricaAdapterImpl.init() in YandexMobileAds
```

Verified present in the 8.3.0 and 8.4.0 binaries — assume every 8.x has it.

In Unity the trigger is **Unity 6000.3**, whose iOS trampoline sets `BUILD_LIBRARY_FOR_DISTRIBUTION = YES` on the `UnityFramework` target (6000.0 does not), and CocoaPods copies that from any user target onto *every* pod target ([`analyzer.rb:455-461`](https://github.com/CocoaPods/CocoaPods/blob/master/lib/cocoapods/installer/analyzer.rb#L455)). Yandex Ads 7.x is unaffected — it reached the adapter through ObjC (`@objc(AMAAnalyticsLibraryAdapter)` / `sharedInstance`), which carries no Swift ABI.

Fix is on the build side, not the version side: clear the flag in a `[PostProcessBuild(Int32.MinValue)]` hook (before EDM4U's `pod install`) on the project and both Unity targets, or force `BUILD_LIBRARY_FOR_DISTRIBUTION = NO` on the AppMetrica pod targets in a Podfile `post_install`. Raise this whenever someone pairs Yandex Ads 8.x with Unity 6000.3+, and refuse to sell a version bump as the cure.

---

## Green pairs

Android `prefers` matched exactly **and** the iOS pod constraint satisfiable. Adapter `A.B.C.D` → Yandex SDK `A.B.C`, so the fourth segment is free.

| Yandex adapter | AppMetrica Unity | Android | iOS |
|---|---|---|---|
| `7.5.0.x` | `6.3.0` | 7.2.0 = 7.2.0 ✅ | 5.8.0 ∈ `[5.0.0,6.0.0)` ✅ |
| `7.13.0.x` / `7.14.0.x` / `7.14.1.x` | `6.5.0` / `6.5.1` / `6.5.2` | 7.9.0 = 7.9.0 ✅ | 5.11.0 ∈ `[5.10.0,6.0.0)` / `[5.11.0,6.0.0)` ✅ |

That is the complete list as of this snapshot. **Two.** If a user is on anything else, they are on a compromise, and the honest answer names which compromise.

### Android-green, iOS-broken

Real pairs, and the trap this skill exists to catch. Fine for Android-only projects; a `pod install` failure otherwise.

| Yandex adapter | AppMetrica Unity | Why iOS fails |
|---|---|---|
| `7.16.0.x` / `7.16.1.x` | `6.7.0` | Yandex wants `~> 5.12.1` (`< 5.13.0`); plugin pins 5.13.0 |
| `8.0.0.x` | `6.8.0` | Yandex wants `~> 6.0.0`; plugin pins 5.16.0 |
| `8.1.0.x` | `6.9.0` | Yandex wants `~> 6.3.0`; plugin pins 6.2.0 |

### Best available compromise on the 8.x line

| Yandex adapter | AppMetrica Unity | Android | iOS |
|---|---|---|---|
| `8.2.0.x` / `8.3.0.x` | `6.10.0` | 8.4.1 vs `prefers` 8.3.0 ⚠️ inside `strictly [8.3.0,9.0.0)` | `~> 6.3` resolves to the 6.4.x/6.5.x Yandex asks for ✅ |
| `8.1.0.x` | `6.10.0` | 8.4.1 vs `prefers` 8.1.0 ⚠️ | ✅ |
| `8.4.0.x` / `8.5.0.x` | `6.10.0` | 8.4.1 is **below** `strictly [8.5.0,9.0.0)` ❌ — Gradle silently raises native to 8.5.0 | `~> 6.3` covers the 6.6.x Yandex asks for ✅ |
| `8.6.0.x` | `6.10.0` | 8.4.1 is **below** `strictly [8.5.1,9.0.0)` ❌ — Gradle silently raises native to 8.5.1 | `~> 6.3` covers the 6.7.x Yandex asks for ✅ |

Not green. `8.2.0.x` / `8.3.0.x` is the better of the two shapes: Android sits one minor *above* `prefers`, so the plugin's wrapper runs the native it was built for. From `8.4.0.x` on, the plugin sits *below* the floor and its C# wrapper drives a native it was never built against — newer adapters make the skew worse, not better. Present `8.3.0.x` + `6.10.0` as the trade-off for anyone who needs 8.x, and say plainly that a clean pairing needs an AppMetrica Unity release shipping native Android 8.3.0 (for 8.2/8.3), 8.5.0 (for 8.4/8.5) or 8.5.1 (for 8.6).

### Adapter versions with no matching plugin at all

`7.6.0`, `7.7.0`–`7.9.0`, `7.10.0`–`7.12.0`, `7.15.x`, `7.17.0`, `7.18.x`, `8.2.0`, `8.3.0`, `8.4.0`, `8.5.0`, `8.6.0` — their `prefers` values (7.2.1, 7.3.0, 7.6.0, 7.10.0, 7.13.0, 7.14.x, 8.3.0, 8.5.0, 8.5.1) are natives no AppMetrica Unity release ships. Anyone on one of these is on an untested combination whether they know it or not; route them to the nearest green pair.

The widest gap to watch for is `7.10.0`–`7.12.0` against plugin `6.7.0`: `prefers` 7.6.0 versus a shipped 7.12.0 is five minors, and it sits inside `strictly [7.6.0,8.0.0)`, so nothing surfaces it.

## Table A

**AppMetrica Unity plugin → the native versions it declares.** From each tag's `Editor/AppMetricaDependencies.xml`.

| Unity plugin | Android SDK (exact spec) | iOS pod |
|---|---|---|
| 6.0.0 / 6.0.1 | 6.4.0 | 5.2.0 (pin) |
| 6.1.0 | 6.5.0 | 5.3.2 (pin) |
| 6.2.0 | 7.0.0 | 5.6.0 (pin) |
| 6.3.0 | 7.2.0 | 5.8.0 (pin) |
| 6.4.0 | 7.7.0 | 5.9.0 (pin) |
| 6.5.0 / 6.5.1 / 6.5.2 | 7.9.0 | 5.11.0 (pin) |
| 6.6.0 | 7.11.0 | 5.12.1 (pin) |
| 6.7.0 | 7.12.0 | 5.13.0 (pin) |
| 6.8.0 | 8.0.0 | 5.16.0 (pin) |
| 6.9.0 | 8.1.0 | 6.2.0 (pin) |
| 6.10.0 | 8.4.1 | **`~> 6.3` (range)** |

Upgrade hazards to raise when recommending a jump:
- **6.4.0** fixed "statistics (installations and new users) when using the AppMetrica Plugins and Yandex Ads Plugins at the same time". Below it, the combination is broken by construction.
- **6.8.0** removed `AppMetrica.reportReferralUrl`, dropped the `AndroidPlayer` module, switched networking to OkHttp, added `setAdvIdentifiersTracking`. Grep the project for `reportReferralUrl` before recommending 6.8.0+.
- **6.10.0** switched the iOS pods to a range — the only reason 8.x is usable at all.

## Table B

**Yandex Ads → what it requires of AppMetrica**, both platforms, straight from package metadata. The Android column is `prefers` (match exactly); the iOS column is the `AppMetricaCore` constraint (must contain the plugin's pinned pod).

| Yandex Ads | Android `prefers` | Android `strictly` | iOS `AppMetricaCore` | Unity plugin that matches Android |
|---|---|---|---|---|
| 7.5.0 | 7.2.0 | `[7.2.0]` | `>= 5.0.0, < 6.0.0` | 6.3.0 ✅ |
| 7.6.0 | 7.2.1 | — | `>= 5.8.0, < 6.0.0` | — |
| 7.7.0 – 7.9.0 | 7.3.0 | — | `>= 5.8.0, < 6.0.0` | — |
| 7.10.0 – 7.12.0 | 7.6.0 | `[7.6.0,8.0.0)` | `>= 5.8.0, < 6.0.0` | — |
| 7.12.1 – 7.12.3 | 7.7.2 | `[7.6.0,8.0.0)` | `>= 5.8.0, < 6.0.0` | 6.4.0 is 7.7.0 ⚠️ patch off |
| 7.13.0, 7.14.0 | 7.9.0 | `[7.9.0,8.0.0)` | `>= 5.10.0, < 6.0.0` | 6.5.x ✅ |
| 7.14.1 | 7.9.0 | `[7.9.0,8.0.0)` | `>= 5.11.0, < 6.0.0` | 6.5.x ✅ |
| 7.15.0 – 7.15.2 | 7.10.0 | — | `>= 5.11.0, < 6.0.0` | — |
| 7.16.0, 7.16.1 | 7.12.0 | `[7.12.0,8.0.0)` | `~> 5.12.1` | 6.7.0 (Android only) |
| 7.17.0 | 7.13.0 | `[7.13.0,8.0.0)` | `~> 5.14.0` | — |
| 7.18.0 – 7.18.2 | 7.14.0 | `[7.14.0,8.0.0)` | `~> 5.14.0` | — |
| 7.18.3 – 7.18.5 | 7.14.2 | `[7.14.2,8.0.0)` | `~> 5.14.0` | — |
| 7.18.6, 7.18.7 | 7.14.3 | `[7.14.3,8.0.0)` | `~> 5.14.0` | — |
| 8.0.0 | 8.0.0 | `[8.0.0,9.0.0)` | `~> 6.0.0` | 6.8.0 (Android only) |
| 8.1.0 | 8.1.0 | `[8.1.0,9.0.0)` | `~> 6.3.0` | 6.9.0 (Android only) |
| 8.2.0, 8.3.0 | 8.3.0 | `[8.3.0,9.0.0)` | `~> 6.4.0` / `~> 6.5.0` | — |
| 8.4.0, 8.5.0 | 8.5.0 | `[8.5.0,9.0.0)` | `~> 6.6, >= 6.6.0` | — (6.10.0 ships 8.4.1, **below the floor**) |
| 8.6.0 | 8.5.1 | `[8.5.1,9.0.0)` | `~> 6.7, >= 6.7.0` | — (6.10.0 ships 8.4.1, **below the floor**) |

Reminder on `~>`: `~> 5.12.1` is `>= 5.12.1, < 5.13.0`, and `~> 6.3` (no patch) is `>= 6.3, < 7.0`. The missing patch segment is the whole difference between "one minor" and "one major".

Note the 7.15.2, 7.18.5 and 7.18.7 rows have no CocoaPods release — those Android-only versions inherit the iOS constraint shown.

---

## Refreshing this file

Do this whenever a version you were asked about is missing from the tables — not only when it is newer than the snapshot. The script below regenerates Table B end to end; it is the authoritative path and takes under a minute.

```bash
python - <<'PY'
import json,urllib.request,re
meta=urllib.request.urlopen("https://repo1.maven.org/maven2/com/yandex/android/mobileads/maven-metadata.xml",timeout=30).read().decode()
vs=[v for v in re.findall(r"<version>([^<]+)</version>",meta) if re.fullmatch(r"\d+\.\d+\.\d+",v)]
print(f"{'yandex':10} {'prefers':10} {'strictly':22} ios AppMetricaCore")
for v in vs[-25:]:
    pref=strict=None
    try:
        d=json.load(urllib.request.urlopen(f"https://repo1.maven.org/maven2/com/yandex/android/mobileads/{v}/mobileads-{v}.module",timeout=30))
        for var in d.get("variants",[]):
            for dep in var.get("dependencies",[]) or []:
                if dep.get("module")=="analytics":
                    ver=dep.get("version") or {}
                    pref=ver.get("prefers") or ver.get("requires"); strict=ver.get("strictly"); break
            if pref: break
    except Exception: pref="?"
    try:
        p=json.load(urllib.request.urlopen(f"https://raw.githubusercontent.com/CocoaPods/Specs/master/Specs/a/c/5/YandexMobileAds/{v}/YandexMobileAds.podspec.json",timeout=30))
        ios=(p.get("dependencies") or {}).get("AppMetricaCore")
    except Exception: ios="(no pod)"
    print(f"{v:10} {str(pref):10} {str(strict):22} {ios}")
PY
```

The CocoaPods path shard `a/c/5` is `md5("YandexMobileAds")[:3]`; for another pod recompute it. `AppMetricaAnalytics` is `c/6/6`.

**Table A** — the plugin's own declarations per tag:

```bash
for t in v6.9.0 v6.10.0 v6.11.0; do echo "== $t"; gh api "repos/appmetrica/appmetrica-unity-plugin/contents/Editor/AppMetricaDependencies.xml?ref=$t" -q '.content' | base64 -d | grep -E "androidPackage|iosPod"; done
```

Release notes for upgrade hazards (removed APIs, behaviour changes):

```bash
gh api repos/appmetrica/appmetrica-unity-plugin/releases --paginate -q '.[] | "== \(.tag_name) ==\n\(.body)"'
```

**Newest AppLovin adapter** — the Maven repo blocks listing, so ask AppLovin's integration manager:

```bash
curl -sL "https://max.applovin.com/docs/v1/unity_integration_manager?plugin_version=8.6.4" -o im.json && python -c "
import json
d=json.load(open('im.json'))
def walk(o):
    if isinstance(o,dict):
        if o.get('Name')=='YANDEX_NETWORK': print(o.get('LatestVersions'))
        for v in o.values(): walk(v)
    elif isinstance(o,list):
        for v in o: walk(v)
walk(d)"
```

To confirm a specific adapter version exists, request the `.aar` (the `.pom` returns 403, the `.aar` does not):

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://artifacts.applovin.com/android/com/applovin/mediation/yandex-adapter/7.16.0.0/yandex-adapter-7.16.0.0.aar"
```

Human-readable changelogs, useful for *why* a version changed but not as the source of truth for numbers: [Yandex Ads Android](https://ads.yandex.com/helpcenter/en/dev/android/changelog-android), [Yandex Ads iOS](https://ads.yandex.com/helpcenter/en/dev/ios/changelog-ios).
