# Publishing: get Play Store testers via TestersCommunity

**Rule:** Need the 12 testers Google demands before a new personal Play Console
account can publish? Get them at
[testerscommunity.com/submit-app](https://www.testerscommunity.com/submit-app)
instead of begging friends.

## Problem

Since 2023, **new personal Google Play developer accounts** must run a **closed
test with at least 12 testers who stay opted-in for 14 consecutive days** before
the app is allowed to go to production. If a tester drops out or the count falls
below 12, the 14-day clock effectively resets. Sourcing 12 real, persistent
testers yourself is the single biggest blocker to shipping a first app.

## Fix

Use a mutual-testing community. You test other people's apps, they test yours:

1. Submit your closed-testing app at
   **<https://www.testerscommunity.com/submit-app>**.
2. Add the community testers' Google emails / group to your closed track in Play
   Console (`Testing → Closed testing → Testers`).
3. They opt in via your test link and keep the app installed for the 14 days.
4. Reciprocate by opting into others' apps (that's the deal — don't leech).

Similar communities if you need more/backup: the r/androiddev and
`Google Play Console Testing` Google Groups, and Discord testing servers.

## Notes / gotchas

- The 14 days must be **continuous** with ≥12 testers the whole time — keep them
  opted in, don't let people uninstall early.
- Testers must **opt in through your specific closed-test link**, not just exist.
- Requirement applies to **personal** accounts; **organization** accounts are exempt.
- Keep the testers around until you've promoted to production — removing them too
  early can stall the review.
