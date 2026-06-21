---
name: android
description: |
  MUST be invoked before any work involving: the `kind: android` schema kind, a `target: android` deploy, the `apk:` layer package format (installing Android apps declaratively), AndroidDeployTarget, an in-pod emulator OR a remote/physical adb-endpoint device, or nested `pod → android` deployment. The first-class Android device + app surface that sits above `charly check adb`/`appium`.
---

# kind: android + the `apk` package format

## Overview

Android is a first-class deploy **substrate** in `charly`, modeled on `kind: k8s`.
Two cooperating concepts:

- **`kind: android`** — a DEVICE (the substrate). Either an in-pod emulator
  (referenced by `image:`) or a remote/physical adb endpoint (`adb:
  {host: <host:port>}`). The analogue of `kind: k8s` (the cluster).
- **`apk` package format** — Android apps declared in LAYERS (NOT a kind),
  parallel to `package:`/`aur:`. A `target: android` deploy applies its layers'
  `apk:` packages onto the device. The app is the deployable workload, the way a
  `candy:` image (a `candy:` node carrying `base:`/`from:`) is the workload for a pod/k8s deploy.

This sits ABOVE the device-interaction verbs: `charly check adb` (`/charly-check:adb`)
and `charly check appium` (`/charly-check:appium`) drive a running device; `kind: android`
+ `target: android` declaratively describe a device and the apps installed on
it. The install machinery is shared — see "One installer (R3)".

## `kind: android` — the device

```yaml
# charly.yml — each device is its own name-first entity (the android: discriminator
# holds scalars; non-scalar fields become child nodes)
pixel9a-36:                          # in-pod emulator device
    android:
        box: android-emulator       # `box:` source field → the `candy:` image that BAKES the emulator + system image
        device: pixel_9a             # informational (documents the baked AVD profile)
        api_level: 36                # informational (the API level is a BUILD property of image:)
    pixel9a-36-google_account:       # non-scalar field → child node
        google_account:              # credential-store secret-key refs for apkeep google-play
            email_secret: GOOGLE_ACCOUNT_EMAIL
            token_secret: GOOGLE_AAS_TOKEN

my-phone:                            # remote/physical device
    android:
        serial: 192.168.1.50:5555
    my-phone-adb:                    # non-scalar field → child node
        adb:
            host: 192.168.1.50:5555  # an adb SERVER addr (a host running `adb connect`)
```

**Device source is exclusive: `image:` XOR `adb:`.**

- **`image:` (in-pod)** — apkeep runs INSIDE the emulator pod (`engine exec`);
  the pod's own adb installs onto its emulator. Needs nothing extra on the host.
- **`adb:` (remote/physical)** — apkeep runs ON THE HOST; the APK is pushed to
  the endpoint via goadb (or `adb -H <host> -P <port>` for the apkeep-download
  path). The host needs `android-tools` (adb) + apkeep for the `package:` path
  (PKGBUILD optdepends; apkeep ships as an upstream binary — no Arch package);
  the committed-`apk:` path needs neither (pure goadb push).

**Build-vs-runtime boundary (load-bearing).** The Android system image + API
level are baked into the referenced `candy:` image at BUILD time (sdkmanager in
the android-sdk layer). `kind: android` REFERENCES that image — it never drives
a build. `device:`/`api_level:` are documentation, not assertions or build
drivers. Two API levels = two images, each with its own `kind: android`.

## `apk:` — the package format (declared in layers)

```yaml
# candy/my-android-apps/charly.yml — name-first: the candy name is the top-level key,
# the apk: list is a child node
my-android-apps:
    candy:
        version: 2026.145.1700
    my-android-apps-apk:
        apk:
            - package: org.fdroid.fdroid   # apkeep download by id
              source: apk-pure             # apk-pure(default) | google-play | f-droid | huawei-app-gallery
              arch: x86_64                 # native ABI (apk-pure) — match the emulator
            - apk: tests/data/MyApp.apk    # committed local APK (project-root- or layer-relative), goadb push
```

Each entry is **`package:` (apkeep) XOR `apk:` (committed file)**. `source:`
applies only to `package:`. The format compiles to an `ApkInstallStep` that
ONLY `target: android` executes — at image build (OCITarget) and on
local/vm/pod/k8s targets it is recorded-skipped (there is no Android device
there; this is the same "wrong-venue skip" `aur:` uses off-Arch). A layer
carrying only `apk:` is valid install content.

Compared to a sideloaded APK: apps with a static-library dependency (e.g.
Chrome's Trichrome) can't be sideloaded — install those via the Play Store
image (preinstalled) instead.

## `target: android` deploy

```yaml
# charly.yml — the device deploys INTO the emulator pod (pod → android), so it is a
# resource node placed UNDER the android-stack deploy (was nested:)
android-stack:
    pod:
        image: android-emulator
    device:                              # deploy-into: applies apk: layers onto the device
        android:
            from: pixel9a-36             # → the kind: android device, deployed onto the emulator pod
        device-add_candy:
            add_candy:
                - my-android-apps        # layers whose apk: packages install onto the device
```

`charly bundle add android-stack.device` resolves the device, gates on
`sys.boot_completed`, and installs the `add_candy:` candies' `apk:` packages via
`AndroidDeployTarget`. Apps ride in on `add_candy:` (the same overlay mechanism
local/vm targets use) — there is no separate apk-list field. `charly bundle del`
best-effort `pm uninstall`s each `package:` id (the device/pod lifecycle is
owned by the pod deploy).

A top-level (non-nested) `target: android` deploy works too — for an `image:`
device it resolves the running container by image name; for an `adb:` device it
talks straight to the endpoint.

## Nested deployment (`pod → android`)

Mirrors `vm → k8s`. The emulator runs in a pod; the device deploys onto it; the
apps deploy onto the device. `target: android` is a **passthrough** hop in the
deploy chain (the device shares its host pod's adb venue / the endpoint addr —
there is no shell venue to "enter"), so `charly check live pod.android` runs the
device's checks against the pod's published adb port. A pod's children can only
deploy AFTER `charly start` (the container doesn't exist at `charly bundle add` time),
so `charly bundle add --node-only` brings the pod up first and the children deploy
afterwards by dotted path; `charly check run <bed>` automates this (deploy pod →
config → start → deploy nested children → check-live).

## One installer (R3)

`charly/android_install.go` holds the SINGLE install path. `AndroidDevice`
abstracts where work runs:

- `InstallByPackage` — apkeep download + adb install. In-pod (`engine exec`,
  apkeep baked in the image) for an image device; host (`apkeep` + `adb -H -P`)
  for an endpoint device. google-play creds come from the container env in-pod,
  or the credential store (via `google_account:`) on the host.
- `InstallFromHostApk` — committed APK pushed via goadb, venue-agnostic.

Both `charly check adb install-app` and `charly check adb install` are thin wrappers over
this — so the apk format, the check verbs, and the deploy target can never drift
on single/split/`.xapk` handling.

## R10 bed

`check-android-emulator-pod` nests two devices: `device` (in-pod, installs
F-Droid via apkeep) and `device-net` (the same emulator addressed as a remote
adb ENDPOINT, installs the committed ApiDemos via goadb) — proving both device
kinds with no physical hardware. The android-emulator-layer check ASSERTS the
results (`apk-fdroid-present`/`-launch`, `apk-net-apidemos-present`).

## Implementation map

- `charly/android_spec.go` — `AndroidSpec` / `AndroidAdbEndpoint` /
  `AndroidGoogleAccount` / `ApkPackageSpec`.
- `charly/android_install.go` — `AndroidDevice` + the shared installer.
- `charly/android_target.go` — `AndroidDeployTarget` (consumes the IR).
- `charly/unified_targets_apk.go` — `AndroidUnifiedTarget.Add`/`.Del` (the android
  deploy + teardown logic, reached via `ResolveTarget`).
- `charly/android_deploy_cmd.go` — `findAndroidSpec` + the device-resolution helpers
  (`resolveAndroidDevice`, `androidApkPackageIDs`).
- `charly/install_plan.go` — `ApkInstallStep`; `charly/install_build.go` —
  `compileApkStep`.
- `charly/unified.go` — loader wiring (mirrors every `k8s` site).
- `charly/deploy.go` `BundleNode.Android`; `charly/deploy_add_cmd.go` dispatch +
  `--node-only`; `charly/deploy_chain.go` / `charly/deploy_tree.go` passthrough.

## Related skills

- `/charly-check:adb` — low-level device control (the verbs the installer shares).
- `/charly-check:appium` — UI automation against the device.
- `/charly-core:deploy` — `target: android` is one of the deploy targets.
- `/charly-image:layer` — the `apk:` field is part of the layer schema.
- `/charly-internals:install-plan` — the IR `ApkInstallStep` plugs into.
- `/charly-kubernetes:kubernetes` — the `kind: k8s` pattern `kind: android` mirrors.

## When to Use This Skill

**MUST be invoked** for any task involving `kind: android`, `target: android`,
the `apk:` layer package format, `AndroidDeployTarget`, an adb-endpoint device,
or nested `pod → android` deployment. Invoke BEFORE reading the android_*.go
source or editing a device's `charly.yml` / a layer's `apk:` list.
