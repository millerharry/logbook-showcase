# Postmortem: signing in reset a real profile

**Severity:** P0 (data loss) · **Found:** 1 November 2026, on TestFlight, nine days before launch · **Fixed:** 2 November 2026 · **Users affected:** pre-launch test accounts only

## Summary

After creating and deleting a test account on a TestFlight build, I signed back into my real Apple account. The app greeted me as "Logbook athlete": no photo, zero followers. The profile in Firestore had been overwritten with new-account defaults, and its `createdAt` was that night.

This was the worst kind of bug for a social training app: it silently destroys someone's identity and social graph, and it would have hit any returning user who signed in again.

## Timeline

| When | What |
|---|---|
| 1 Nov, evening | Found on TestFlight. The profile document showed `createdAt` from minutes earlier, a default name, a blank photo link, and empty follower and following lists. |
| 1 Nov | Traced: only one function in the codebase writes `createdAt` to a profile. That narrowed the search to a single code path. |
| 2 Nov | Fix merged. Verified on a device build: three sign-ins plus a delete-and-recreate cycle, and name and photo held. |
| 2 Nov | Turned on Firestore point-in-time recovery and scheduled backups. |

## Root cause

`createUserProfile` ran on **every** Apple sign-in, not just new ones. It protected existing accounts with a check: read the user document, and if it already has identity fields, leave it alone.

The read was a plain `getDocument()`, which **can answer from the local cache**. Several other subsystems (notification preferences, challenge state, weekly goals) write into the same document with `merge: true` as soon as a session exists. So the cached snapshot could hold only another subsystem's pending write, with no identity fields. The guard concluded "no profile here" and wrote a full set of defaults over the real account.

**What I couldn't prove:** the exact cache state was never reproduced. A cached or pending snapshot is the most likely cause, and it matches an earlier, similar issue in the same file. I chose a fix that's correct *whatever* the read returns, rather than one that depends on that explanation being right.

### The irony

That guard was itself the fix for an earlier bug. Originally the function only checked whether the document *existed*. Subsystems often created a stub document first, so new users never got a profile written at all. The fix changed the check from "exists" to "has identity fields" and completed stubs with defaults. That was correct for new users, but it opened a path for returning users whenever the read lied.

## The fix

**1. Read from the server, or don't write at all.** Profile creation now runs inside a Firestore **transaction**. Transaction reads always go to the server and ignore local state, and offline they fail, so an offline sign-in writes nothing instead of guessing.

**2. Creation can only add, never replace.** A small, pure function computes the fields the document is *missing*, and only those are written. Whatever a profile already holds (name, photo, XP, followers) stays. Because it's pure, it's trivial to unit-test.

```swift
static func fieldsMissing(from existing: [String: Any]?, in defaults: [String: Any]) -> [String: Any] {
    guard let existing else { return defaults }
    return defaults.filter { existing[$0.key] == nil }
}
```

**3. Don't run creation when you don't need it.** Returning Apple sign-ins now skip profile creation entirely and go straight to loading the profile. Only an account Apple reports as new gets a profile written.

**4. Recover from transient failures.** A failed profile load used to wait for an auth event that might never come, which left a signed-in user nameless until they relaunched the app. It now retries after 2 s and 5 s.

## Data recovery (the painful part)

The overwritten values **could not be recovered.** Firestore had point-in-time recovery turned off and no backups. I had treated backups as a launch-week chore, and this incident showed they're a precondition for having real users at all.

Now in place:
- point-in-time recovery with 7-day retention
- daily backups kept for 7 days
- weekly backups kept for 14 weeks

## What went well

- **Found before launch**, by testing the "boring" flows (delete, recreate, sign back in) instead of only the happy path.
- **Fast narrowing:** a field written in exactly one place (`createdAt`) was the best clue.
- **The fix doesn't depend on the unproven theory.** It's safe whether the read came from the cache or the server.

## What I changed in how I work

- **Writes that create should be additive by construction**, not protected by a read-then-decide check.
- **Never let a write depend on a read that may have come from the cache.** Use a transaction or require a server read.
- **Backups and point-in-time recovery go on before the first real user**, not after the first incident.
- **Record what's verified and what isn't.** Offline sign-in was still untested after the fix, and I wrote that down rather than calling it "fixed."
