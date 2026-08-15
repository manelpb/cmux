Symptom: After the newly signed-in Mac appeared immediately, reconnect cleared all workspaces, marked all computers disconnected, and left refresh spinning indefinitely.
User impact: Fast discovery succeeds, but reconnect destroys the usable workspace list and cannot recover in-app.
Source: User report and screenshot at 2026-08-14 11:14 local time, plus live iPhone and tagged-Mac logs copied into this directory.
Target surface: iOS physical device Aziz, bundle dev.cmux.ios.rtlist, with tagged Macs rta/rtb/rtlist.
Build/version/tag: rtlist. Incident source bf9d72fac3; lifecycle fix fe8b6e2057.
Repro workload: Sign in to a compatible tagged Mac, wait for immediate discovery, trigger reconnect to fetch new workspaces.
Expected bad behavior: Computer status becomes Not Connected, workspace rows disappear, and the central refresh spinner never settles.

Workload:
1. Start signed in with the Mac and workspaces visible.
2. Trigger reconnect to fetch new workspaces.
3. Observe computer status, workspace retention, and refresh completion.
Stop condition: Logs identify the pending operation/owner, or the same workload completes with the list preserved after a fix.

Evidence:
- The foreground workspace list succeeded in 34 ms at 18:12:07.793.
- The same refresh awaited secondary Iroh discovery until 18:16:02.120, failing after 234.361 seconds.
- A route revision awaited physical QUIC close for more than 224 seconds, delaying policy publication and producing admission denials on replacement attempts.
- The first disconnect cleanup retained the tagged foreground rows. A second cleanup ran after foreground identity had cleared, derived the anonymous key, and filtered those rows out.

Fix:
- Pull-to-refresh now awaits only the authoritative foreground list and schedules secondary discovery under its existing background owner.
- Disconnect cleanup retains `foregroundOrRecoveryMacKey`, which preserves the exact device and app-instance tag across repeated cleanup.
- Route changes logically retire the old peer immediately. Physical close drains in the background, and replacement dials use the existing bounded drain fence.

Verification:
- Regression-only commit 668a76bebf failed all three intended assertions on the remote M4.
- Fix commit fe8b6e2057 passed all three regressions on the remote M4.
