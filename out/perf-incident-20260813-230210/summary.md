Symptom: The rtlist iOS app crashed after rtb appeared in the computer list, then crashed again after reopening it and launching Claude from two terminals.
User impact: The live multi-Mac computer list is visible, but normal terminal use terminates the iOS app.
Source: User report, four iPhone crash reports, iOS application logs, and an isolated regression test.
Target surface: iOS physical device, Aziz iPhone 17 Pro Max.
Build/version/tag: dev.cmux.ios.rtlist, fixed source bf9d72fac3+, signed archive exported 2026-08-13 23:53 local time; physical install is queued while Aziz is unreachable.
Repro workload: Keep rtb online, reopen rtlist, open two terminals, and launch Claude in both.
Expected bad behavior: The rtlist iOS process terminates during or shortly after terminal activity.

Workload:
1. Start with rtlist authenticated and the rtb Mac visible.
2. Open two terminal surfaces and launch Claude in both.
3. Observe whether the iOS process terminates.
Stop condition: A crash report identifies the failing thread, or the workload completes without termination after a fix.

Finding: `cmux-2026-08-13-230643.ips` and `cmux-2026-08-13-230759.ips` fault on the `com.apple.NSURLSession-delegate` queue in `CheckedContinuation.resume(throwing:)`, called by `PresenceSyncTransport.sendPingOnce(_:)`. The process terminates with `EXC_BREAKPOINT` / `SIGTRAP` because the same ping callback resumes one checked continuation more than once.
Owner boundary: The callback-to-async adapter in `PresenceSyncTransport`, not Stack sign-in, terminal rendering, Claude, or the Mac connection list.
Fix: `PresencePingResumeGate` performs a synchronous one-bit compare-and-set before continuation settlement. The first callback propagates its result; duplicate callbacks become no-ops. This is a principled ownership fix because it enforces the continuation's exactly-once contract at the legacy callback seam.
Regression: Commit `fcf2b2beff` failed on AWS M4 Pro with signal 5 and `SWIFT TASK CONTINUATION MISUSE`. Commit `bf9d72fac3` passed the identical `PresenceSyncTransportTests` workload with one test in one suite.
Residual observation: A separate canceled chat-history retry storm appeared while collecting logs. It is not on the faulting stack and is excluded from this crash fix.
Verification status: macOS rtlist is running from the fixed source; the focused duplicate-callback regression is green. The first combined iOS job built the device archive but failed on an unrelated x86_64 Simulator link. A device-only archive exported successfully and is queued for Aziz; physical workload verification remains pending reconnection.
