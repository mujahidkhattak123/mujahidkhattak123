# Mujahid Khattak

**AI engineer, Android side and server side.** I build apps where the interesting part is the model — on-device inference, a multi-provider cloud gateway behind it, and the plumbing that decides which one runs and who pays for it.

---

## Background Eraser Photo Cutout

Live on Google Play · [Store listing](https://play.google.com/store/apps/details?id=com.photostudio.aiphotoeditor)

A photo editor built around segmentation and generative editing. Kotlin and Jetpack Compose on the device, FastAPI on the server.

**On-device** — ML Kit Subject Segmentation runs the free cutout entirely on the handset. Nothing uploaded, works offline, costs nothing per run.

**In the cloud** — a provider registry fronting ten model backends, selected per operation and per tier:

| Operation | Endpoint | Models behind it |
|---|---|---|
| Background removal | `/remove-background` | BiRefNet (self-hosted), Bria, PhotoRoom, rembg |
| Generative replace | `/ai-replace` | Replicate, fal, kie |
| Inpainting | `/magic-erase` | Replicate inpaint |
| Upscale | `/enhance` | Replicate upscale |
| Text-to-image | `/generate` | OpenAI Images, fal, kie |

BiRefNet runs on a RunPod GPU worker I built and deploy myself, rather than renting it per call. The rest sit behind one `AIProvider` interface, so swapping a vendor is a config change and a vendor outage is not an outage.

**Around the models** — async generation jobs with polling and reconciliation, credit metering per tier, rate limiting, content moderation, and an in-app reporting flow for AI-generated results.

---

## Four problems worth the time they took

### A readiness check that lied

ML Kit's segmentation model is an optional Play Services module, fetched on first use. ML Kit exposes no "is it ready" API, so the app inferred readiness by segmenting a 64×64 grey square and treating any non-`UNAVAILABLE` outcome as success.

It does not work. Measured on a device:

```
20:25:48  probe says ready, 1198ms
20:27:58  real cutout on a real photo → UNAVAILABLE (code 14)
```

A check that can pass while the thing it checks is absent is worse than no check, because it is what tells the warm-up to stop trying. Replaced with Play Services' own `ModuleInstall` API — `areModulesAvailable` answers truthfully, and `installModules` actually *starts* the download, which the probe never did. First cutout went from "still downloading" to **1.2 seconds**.

### Choosing an engine without lying to the user

Paid cloud removal gives better edges; the on-device model is free and instant once present. The rule that decides between them has been reversed twice in this codebase, plausibly each time.

It now lives in one pure function with no I/O, pinned by tests: **escalate free to paid, never the reverse.** A standard request is promoted to the cloud only when the on-device module has not yet proven itself *and* the user already owns the paid tier — so being routed to the better engine can never surface a paywall. A paid request is never quietly downgraded; the failure is surfaced with the choice left to the user.

### Withdrawing a generative feature without deleting it

A Play policy enforcement required removing all AI-generated content from a live app, quickly. Deleting the code would have made restoring it a rewrite.

Instead: one compile-time flag, nineteen guards, and a router-level dependency returning 503. Every entry point — home, editor, menus, search, deep links, saved projects — resolves through the flag, so no path reaches generation however it was started. A test asserts the route stays mounted but refuses. Restoring the feature is one boolean.

The same enforcement needed in-app reporting for AI results. That ships: reason taxonomy, optional detail, Firestore-backed, and deliberately **no auth tokens, no credentials, no report text in logs** — a uid prefix is all that is recorded.

### Measuring before building

GA4 said 2,562 installs reached 1,041 editor sessions and 49 completed exports. People were spending **two minutes** in the editor and leaving with nothing — not a discovery problem, a completion problem.

Two causes found and fixed: a paywall sitting inside the removal flow, and the model download above. The third was not a code problem at all — two of five live ad headlines promised a PNG converter the app did not have. So I built the converter.

---

## Stack

**On-device ML** — ML Kit (subject segmentation, face detection), model lifecycle and Play Services optional modules
**Model serving** — FastAPI, provider abstraction over BiRefNet / Replicate / fal / OpenAI Images / Bria / PhotoRoom, self-hosted GPU inference on RunPod, async job orchestration
**Android** — Kotlin, Jetpack Compose, Coroutines/Flow, Room, CameraX, Play Billing, Play App Signing, AdMob, UMP
**Infra** — Docker, Railway, Firebase Auth, Firestore, GitHub Actions
**Practice** — on-device measurement before changing behaviour, unit tests that pin defects with numbers, feature flags over branches

---

## Contact

📧 [mujahidmskhan112@gmail.com](mailto:mujahidmskhan112@gmail.com)
