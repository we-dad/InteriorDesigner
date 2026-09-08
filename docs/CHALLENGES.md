# Technical Challenges

Problems encountered building Space Genie, and how they were resolved.

## 1. Crash on designer cards with an empty portfolio

**Symptom**
The home feed crashed with `Index out of range` as soon as a newly registered designer appeared in it. It worked throughout development and broke the moment a real account signed up without uploading work first.

**Diagnosis**
`home.swift` read the designer's cover image with a direct subscript:

```swift
let im = d.images[0]
AsyncImage(url: URL(string: "\(im)"))
```

`images` is populated from Firestore and is empty for any designer who has completed signup but not yet built a portfolio. The field had been modelled as always-present because every seeded test account had images — but the signup flow never required them.

**Alternatives considered**

| Option | Outcome |
|---|---|
| Require at least one image at signup | ❌ Rejected — adds friction to the hardest step of a two-sided marketplace, where designer supply is the bottleneck |
| Filter empty-portfolio designers out of the feed | ❌ Rejected — hides new designers from clients, the opposite of what the marketplace needs |
| Render a placeholder when the array is empty | ✅ Chosen |

**Fix**
Guard the subscript and fall back to a neutral placeholder with identical dimensions, so the feed layout doesn't shift:

```swift
if d.images.count == 0 {
    Color.gray.frame(width: 327, height: 154).cornerRadius(4)
} else {
    AsyncImage(url: URL(string: d.images[0])) { ... }
}
```

Commit `b20d346` — *solve error index*

**Lesson**
Seed data is not test data. Every field read from a remote document is optional until the write path structurally guarantees otherwise, and the signup flow guaranteed nothing. The better fix was at the type level — modelling the cover image as `String?` rather than reaching into an array — which would have let the compiler catch this before a user did.

---

## 2. Designing the chat data model: reads or writes?

**Context**
Real-time messaging between clients and designers, with a conversation list showing the latest message per thread. Firestore bills both reads and writes per document, and the conversation list is opened far more often than a message is sent.

**Alternatives considered**

| Model | Trade-off |
|---|---|
| Single `messages` collection, queried by participant | Cheap writes, but the conversation list needs a query per thread plus a composite index; read cost scales with conversation count |
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
Four writes per message and no single source of truth. Deleting or editing means touching four locations, and a partial failure leaves the two copies out of sync — the writes were not wrapped in a transaction. Acceptable for a closed test group; at scale it needs a batched write or a Cloud Function fan-out.

**Attribution**
The messaging module is adapted from Brian Voong's *LBTASwiftUIFirebaseChat*; original file headers are preserved in `FireBaseChat/`. It was extended with media attachments, gender-aware avatars, and integration with the designer profile model.

---

## 3. Sign in with Apple returns the user's name exactly once

**Symptom**
Accounts created through Sign in with Apple showed an empty name on the profile screen. Email/password signup worked correctly.

**Diagnosis**
Apple returns `fullName` only on the **first** authorization for a given Apple ID and app pairing. On every subsequent sign-in it is `nil` by design — Apple treats it as one-time data the app is expected to persist. The field was being read in the sign-in handler on every launch rather than captured at account creation, so any account created before the persistence step existed was permanently nameless.

**Fix**
Capture `fullName` inside the credential callback at first authorization and write it to the Firestore user document immediately, treating Apple's local response — not Firebase Auth — as the only source for that field.

Commit `522e136` — *sign with apple full name*

**Lesson**
Third-party identity providers each have their own contract about what data arrives when. The failure mode here is silent: no error, no crash, just a blank field discovered later. Any one-time value from an external provider has to be persisted the moment it arrives.

## What would be built differently

| Area | Then | Now |
|---|---|---|
| Data modelling | Optional Firestore fields read as non-optional | Optional types at the model boundary |
| Chat writes | Four unwrapped writes | Batched write or Cloud Function fan-out |
| Observability | None | Crash reporting from day one — challenge 1 would have surfaced in minutes |
