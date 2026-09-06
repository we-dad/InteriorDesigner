# Technical Challenges

Engineering problems encountered while building Space Genie (May–Oct 2023), and how they were resolved. Each entry follows the same structure: symptom, diagnosis, alternatives, fix, and what it taught us.

> **On evidence.** This project shipped before we instrumented anything, and the Firebase project was deprovisioned when the academy program ended. We have no production metrics. Rather than invent numbers, the entries below cite the code and commits that document each problem.

---

## 1. Crash on designer cards with an empty portfolio

**Symptom**
The home feed crashed with `Index out of range` as soon as a newly registered designer appeared in it. It worked fine during development and broke the moment a real account signed up without uploading work first.

**Diagnosis**
`home.swift` read the designer's cover image with a direct subscript:

```swift
let im = d.images[0]
AsyncImage(url: URL(string: "\(im)"))
```

`images` is populated from Firestore and is empty for any designer who has completed signup but not yet built a portfolio. We had modelled the field as always-present because every test account in our seed data had images. The signup flow never required them.

**Alternatives considered**

| Option | Outcome |
|---|---|
| Require at least one image at signup | ❌ Rejected — adds friction to the hardest step of a two-sided marketplace, where designer supply is the bottleneck |
| Filter empty-portfolio designers out of the feed | ❌ Rejected — hides new designers from clients, which is the opposite of what the marketplace needs |
| Render a placeholder when the array is empty | ✅ Chosen |

**Fix**
Guard the subscript and fall back to a neutral placeholder that keeps the card's dimensions identical, so the feed layout does not shift:

```swift
if d.images.count == 0 {
    Color.gray.frame(width: 327, height: 154).cornerRadius(4)
} else {
    AsyncImage(url: URL(string: d.images[0])) { ... }
}
```

Commit: `b20d346` — *solve error index*

**Lesson**
Seed data is not test data. Every field read from a remote document is optional until the write path structurally guarantees otherwise, and our signup flow guaranteed nothing. In hindsight the right fix was at the type level — modelling the cover image as `String?` rather than reaching into an array — which would have made the compiler catch this before a user did.

---

## 2. Designing the chat data model: reads or writes?

**Context**
Real-time messaging between clients and designers, with a conversation list showing the latest message per thread. In Firestore, both reads and writes are billed per document, and the conversation list is opened far more often than a message is sent.

**Alternatives considered**

| Model | Trade-off |
|---|---|
| Single `messages` collection, query by participant | Cheap writes, but the conversation list needs a query per thread plus a composite index; read cost scales with conversation count |
| One document per thread with an embedded message array | Simple, but Firestore caps documents at 1 MB and every new message rewrites the whole document |
| Denormalized dual-write | ✅ Chosen — expensive writes, very cheap reads |

**Implementation**
Each sent message writes to four paths:

```
messages/{fromId}/{toId}/{autoId}           sender's copy
messages/{toId}/{fromId}/{autoId}           recipient's copy
recent_messages/{fromId}/messages/{toId}    sender's conversation list entry
recent_messages/{toId}/messages/{fromId}    recipient's conversation list entry
```

The conversation list then costs a single ordered listener on one path, with no composite index required:

```swift
.collection("recent_messages").document(uid).collection("messages")
.order(by: "timestamp")
.addSnapshotListener { ... }
```

Source: `ChatLogView.swift:83–185`, `MainMessagesView.swift:41–45`

**Accepted trade-off**
Four writes per message and no single source of truth. Deleting or editing a message means touching four locations, and a partial failure leaves the two copies out of sync — we had no transaction wrapping the writes. For a closed test group this was acceptable; at scale it would need a batched write or a Cloud Function fan-out.

**Attribution**
The messaging module is adapted from Brian Voong's *LBTASwiftUIFirebaseChat* tutorial; the original file headers are preserved in `FireBaseChat/`. We extended it with media attachments, gender-aware avatars, and integration with our designer profile model.

---

## 3. Sign in with Apple returns the user's name exactly once

**Symptom**
Accounts created through Sign in with Apple showed an empty name on the profile screen. Email/password signup worked correctly.

**Diagnosis**
Apple returns `fullName` only on the **first** authorization for a given Apple ID and app pairing. On every subsequent sign-in the field is `nil` by design — Apple treats it as one-time data the app is expected to persist. We were reading it during the sign-in handler on every launch rather than capturing it at account creation, so any account created during testing before the persistence step existed was permanently nameless.

**Fix**
Capture `fullName` inside the credential callback at first authorization and write it to the Firestore user document immediately, treating the local Apple response — not Firebase Auth — as the only source for that field.

Commit: `522e136` — *sign with apple full name*

**Lesson**
Third-party identity providers each have their own contract about what data arrives when. The failure mode is silent: no error, no crash, just a blank field discovered later. Any one-time value from an external provider must be persisted at the moment it arrives.

---

## 4. Merge conflicts from committed Xcode state files

**Symptom**
On 7 June the repository accumulated nine merge commits in a single day across three developers. A significant share of conflicts were in files nobody had meaningfully edited.

**Diagnosis**
`UserInterfaceState.xcuserstate` was tracked in git. Xcode rewrites this binary file constantly — it records window positions, open tabs, and scroll offsets — so it changes on every developer's machine on every session, and being binary it can never auto-merge. The repository also had no `.gitignore` at all.

**Fix**
Untrack the file and stop committing per-developer Xcode state.

**Lesson**
A `.gitignore` is the first commit of a project, not a cleanup task. On a team of three, tooling noise consumed time that should have gone to features — and unlike a bug, this cost was invisible because it never appeared as a defect.

---

## 5. Portfolio images compressed but never resized

**Symptom**
Portfolio grids scrolled unevenly on older devices, and Storage usage grew faster than expected relative to the number of uploads.

**Diagnosis**
The upload path applies JPEG compression but leaves pixel dimensions untouched:

```swift
guard let uploadData = image.jpegData(compressionQuality: 0.5) else { return }
```

Source: `FirebaseManager.swift:37`

A 12 MP iPhone photo drops from roughly 4 MB to 1 MB, but remains 4032×3024 pixels. Those pixels are then decoded in full and downsampled at render time into a 327×154 card — the memory cost of decoding is driven by dimensions, not file size, so compression alone did not help scrolling performance at all.

**What we would do differently**
Downsample to roughly twice the display size before upload, and store a separate thumbnail for feed cards while keeping the full-resolution original for the detail view. `ImageIO`'s `CGImageSourceCreateThumbnailAtIndex` does this without fully decoding the source image.

**Lesson**
Compression and resizing solve different problems. File size affects bandwidth and storage cost; pixel dimensions affect decode memory and scroll performance. We reached for the first and assumed it addressed the second.

---

## 6. Known limitation: the OpenAI key shipped in the client

The ChatGPT assistant calls the OpenAI API directly from the iOS app, with the key held in a compiled constant (`Constants.swift`) and read in `OpenAIService.swift`. Anything compiled into an app bundle is extractable from the IPA, so this key was never actually secret.

We accepted this for an academy project with a closed test group and a spending cap on the key. It is not acceptable for anything with real users.

**The correct design** is a thin server-side proxy — a Firebase Cloud Function holding the key in environment config, authenticating each caller against Firebase Auth, and rate-limiting per user. The client then calls our own endpoint and never sees a provider credential. This also makes the model swappable without shipping an app update.

**Lesson**
"Secret" in a client application is a category error. If the value must stay private, it has to live somewhere the user does not control.

---

## What we would build differently

| Area | Then | Now |
|---|---|---|
| Data modelling | Optional Firestore fields read as non-optional | Optional types at the model boundary |
| Secrets | Compiled into the client | Server-side proxy |
| Images | Compressed only | Downsampled with separate thumbnails |
| Chat writes | Four unwrapped writes | Batched write or Cloud Function fan-out |
| Repo hygiene | No `.gitignore` | Ignore file in the first commit |
| Observability | None | Crash reporting from day one — challenge #1 would have surfaced in minutes |
