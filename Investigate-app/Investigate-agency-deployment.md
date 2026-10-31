"Same codebase" doesn't mean everyone runs the exact same code at the same moment. It means **one Git repository, with one history, that all versions come from**. A version is just a **snapshot of that repository at a point in time**.

## A simple example

Think of the WhatsApp app. Your phone might have version 2.24 and your friend's phone version 2.25. They're different versions, but it's still **one WhatsApp codebase**. Your friend just has a newer snapshot. WhatsApp didn't build a separate app for each person.

Investigate works the same way:

```
One Git repository (Investigate)
──────●────────●────────●────────●────────●──────→  main (keeps moving forward)
      │                 │
      │                 └─ support/INV_SP270 → tag v2.7.1 → deployed to Pune
      │
      └─ support/INV_SP260 → tag v2.6.2 → deployed to Mumbai
```

- v2.6.2 is the code **as it was** at the time of release 2.6.
- v2.7.1 is the same code **a few months later**, with new features added.
- Both come from **the same repository and the same history**.

## What a "different codebase" would mean

| Same codebase (Investigate) | Different codebase (avoided) |
|---|---|
| One repository | A separate copy of the code per customer |
| Versions are just points in time | Custom code written only for Pune or only for Mumbai |
| A bug fix is made once and goes into every version | The same bug has to be fixed separately in each copy |
| Mumbai will eventually get every 2.7 feature | Mumbai's copy drifts apart and never catches up |
| Differences between agencies come from **configuration** | Differences between agencies are in **the code itself** |

## How the versions stay one codebase

1. **New work always goes to `main`.** Every future version contains it.
2. **Fixes go back to `main` too.** If a bug is fixed for Mumbai in 2.6, the same fix is also merged into 2.7 and `main`, so the versions never become separate products.
3. **Old versions only get bug fixes, no new features.** New features only go forward.
4. **Every agency moves forward eventually.** Mumbai will upgrade to 2.7, then 2.8. Nobody stays on a separate track forever.

## How to say it in the interview

> "Same codebase means one repository and one history. A version is just a snapshot of that code at a point in time. Mumbai on 2.6 is running an older snapshot and Pune on 2.7 a newer one, like two phones with different versions of the same app. There's no custom code per agency; differences come only from configuration. Bug fixes are merged into every supported version and into main, so the versions never split into separate products, and every agency eventually moves forward to the latest release."
