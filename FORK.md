# Why this fork exists

`destinyspace/mediapipe_face_mesh` is a fork of
[`cornpip/mediapipe_face_mesh`](https://github.com/cornpip/mediapipe_face_mesh)
carrying **two changes on one branch**.

| | |
|---|---|
| branch | `destiny/2.4.0-models-trimmed` |
| forked from | upstream tag `v2.4.0` (`3a1bb60b4f2f128e55cc763bb38b2f37af8417dc`) |
| diff vs upstream | `pubspec.yaml` — the `flutter: assets:` list · `ios/` — an iOS privacy manifest and the `resource_bundles` line that ships it |
| consumer | [`destinyspace/destiny`](https://github.com/destinyspace/destiny) · issues NGH-721, NGH-784 |

## Change 1 — the trimmed asset list

Upstream declares the model directory:

```yaml
  assets:
    - assets/models/
```

Flutter bundles every declared asset, so all seven `.tflite` models ship in
every consumer's app whether or not it opens them. destiny opens exactly two —
the base 468-point mesh and the short-range detector — because
`enableIris`, `enableAttentionMesh` and the blendshapes processor are all off.
This branch names those two instead of the directory.

| model | bytes | on this branch |
|---|---:|---|
| `mediapipe_face_mesh.tflite` | 1,242,398 | declared |
| `face_detection_short_range.tflite` | 229,032 | declared |
| `iris_landmark.tflite` | 2,640,568 | not declared |
| `face_landmark_with_attention.tflite` | 2,495,952 | not declared |
| `face_detection_full_range.tflite` | 1,083,786 | not declared |
| `face_blendshapes.tflite` | 955,312 | not declared |
| `face_detection_full_range_sparse.tflite` | 676,746 | not declared |

**7,852,364 bytes** leave the bundle, on Android and iOS alike. The files stay
in the repository; only the declaration narrows, so rebasing onto a new upstream
tag is a two-line replacement.

## Change 2 — an iOS privacy manifest for the pod's own binary

Apple evaluates ITMS-91053 **per Mach-O binary**. `TensorFlowLiteC` ships here
as a *static* Mach-O object (`file` reports "Mach-O 64-bit object", not a
dylib), so it never reaches a built app as a binary of its own — the linker
folds it into `mediapipe_face_mesh.framework`, and that framework is where
`_stat` / `_fstat` are undefined. It calls them to `mmap` the `.tflite` models,
which is Apple's file-timestamp required-reason category.

Upstream TensorFlow declares exactly that, and its declaration has been sitting
in this tree the whole time at
`src/include/tensorflow/lite/ios/TensorFlowLiteC.xcprivacy` — the podspec set
`vendored_frameworks` and no `resource_bundles`, so it was never copied into
any built product. Every consumer therefore had to re-declare it at app level
or be rejected at upload, and an app-level declaration is a claim about the
app's binary, not this one.

| | |
|---|---|
| `ios/Resources/PrivacyInfo.xcprivacy` | new — TensorFlow's `FileTimestamp` / `C617.1` declaration, under the filename Apple looks for |
| `ios/mediapipe_face_mesh.podspec` | new `resource_bundles` entry shipping it as `mediapipe_face_mesh_privacy.bundle` |

The manifest declares `NSPrivacyTracking false` with empty collected-data and
tracking-domain arrays. That is true *for this pod* — inference is on-device,
it opens no network connection — and is a different claim from any consumer
app's, which is why consumers keep their own manifest.

**Rebasing:** this change is additive and touches no upstream file except the
podspec, so it re-applies cleanly. If upstream ever adds its own
`resource_bundles`, merge into that hash rather than replacing it.

## Do not use this branch if you need iris / blendshapes / full-range

`rootBundle.load()` throws for a model that is not declared. Any consumer that
sets `enableIris: true`, `enableAttentionMesh: true`, builds a
`FaceBlendshapesProcessor`, or picks `FaceDetectionModel.fullRange` /
`fullRangeSparse` will fail at load time on this branch. Use upstream, or add
the model you need to the list.

## Upgrading to a new upstream version

1. `git fetch upstream && git checkout -b destiny/<new>-models-trimmed <upstream tag>`
2. Re-apply the `assets:` list. **Re-read `assets/models/` first** — a renamed,
   split or added model changes what a consumer has to declare.
3. Push, then pin the new commit sha in destiny's `frontend/app/pubspec.yaml`.
4. destiny's bundle guard (`test/tuong_so/bundled_models_guard_test.dart`)
   asserts the bundled model set and its exact byte sizes, so a bad re-apply
   goes red there rather than shipping.

Upstream is BSD-3-Clause; the licence and copyright are unchanged.
