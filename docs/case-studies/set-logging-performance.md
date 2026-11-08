# A set logger that feels instant

Logging sets is the core of the product: it's the reason someone picks Logbook over Hevy or Strong. A real session can have 10 exercises × 4 sets × 3 fields, and every keystroke has to feel immediate.

## Targets

| Interaction | Budget |
|---|---|
| Tap a field → keyboard up | < 100 ms |
| Each character typed | < 50 ms |
| Opening a workout list | < 200 ms |

## What goes wrong by default

The obvious SwiftUI approach (bind each `TextField` straight to the model) sets off a chain on every keystroke:

```
keystroke → model mutates → parent view invalidates → whole list re-renders
          → PR detection recalculates → persistence writes
```

With a few hundred fields on screen, that adds up to visible lag.

## The pattern

**1. Local state, then commit.** Each `SetInputRow` owns its text in local `@State`. You type freely, and the model is only touched on commit: blur, Done, or a short typing pause (0.75 s). The pause matters because numeric keypads give no reliable "submit" event.

**2. Debounce the expensive work.** PR detection and validation wait ~300 ms after typing stops. PR data is cached in the service layer, so the check is a lookup, not a scan of your history. The PR badge updates optimistically, and the full recalculation runs in the background when the workout is saved.

**3. Never lose a set.** `WorkoutSessionStore` autosaves the draft about 0.65 s after the last change (a 150 ms debounce plus a 500 ms save delay) and again when the app goes to the background. The JSON write happens off the main thread. If the app is killed mid-workout, Resume brings the session back.

**4. Keep views small.** Large views are split into focused components, specific values are passed instead of whole objects, and conformance to `Equatable` stops rows re-rendering when nothing they show has changed. `Self._printChanges()` is the profiling tool: if a row logs "@self changed" on every keystroke elsewhere, it's wired wrong.

## Guarding it

Performance rules live in the repo alongside the code (`performance-rules.md`, `common-patterns.md`), so every contributor, human or AI agent, starts from the same budgets and patterns. Regressions get caught in review against those budgets rather than rediscovered by users.
