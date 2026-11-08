# Offline-first sync

Gyms are basements and concrete boxes, so the network drops out constantly. A training log that loses a workout because the signal dropped is a log nobody trusts.

## Principle

**The device is where writes happen; the cloud catches up.** Every save lands locally first (Core Data plus JSON caches) and is put on a persisted queue. Firestore is the shared source of truth for profiles, workouts, clubs and feeds, but you never wait on it to keep training.

## The sync queue

`SyncQueueService` holds pending cloud writes (workouts, activities, club posts, club creation) as typed, `Codable` payloads persisted to disk, and retries them when connectivity and authentication allow.

Three details that took real work:

**1. Account isolation.** If you sign out and someone else signs in on the same phone, your queued writes must never be sent as theirs. Each entry is tied to the account that created it, and the current-user lookup is *injected* (`authUidProvider`) so the isolation rules can be unit-tested without Firebase.

**2. Wire formats outlive refactors.** Midway through the project, "Circles" were renamed "Clubs" across the codebase. Queued payloads written by older builds still say `circleId`. Explicit `CodingKeys` keep the old wire format, so a write queued before an update still decodes and sends after it. The Swift names changed; the data contract didn't.

**3. Idempotent writes.** A retry after a timeout must not create a duplicate workout. Each workout document is keyed by its client-generated UUID and written with a full `setData`, so repeating the write just overwrites the same document.

## Security

Firestore security rules are treated as a deployable artifact: version-controlled, diffed against the live ruleset before each deploy, and read back afterwards. Privacy controls (private workouts and profiles, follower-only visibility) are enforced in the rules, not just hidden in the UI.

## Lessons

- **Test the edge cases nobody notices:** a second account on the same device, sign-in while offline, a payload queued two versions ago.
- **Turn on backups and point-in-time recovery before launch, not after.** Recovering from a bad production write is much easier when the history exists.
