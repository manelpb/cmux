# iOS terminal render freeze incident

Symptom: Terminal pixels intermittently stop updating during ordinary cmux use on iOS.
User impact: The terminal appears frozen and can no longer be trusted to show current output.
Source: User report on 2026-08-14, prior freeze PRs, and current open fix PR https://github.com/manaflow-ai/cmux/pull/10125.
Target surface: iOS terminal renderer and its Mac-to-iOS output pipeline.
Build/version/tag: Reported build unknown. Baseline under investigation is `origin/main` at `8033c260b7`; candidate branch head is `76152bfc5c`.
Repro workload: Active terminal output plus local scroll, geometry changes, foreground/background transitions, replay, reconnect, and surface remount.
Expected bad behavior: A frame, replay barrier, output consumer, or viewport transaction remains unresolved, so visible pixels stay stale while later work is blocked or hidden.

## Repeatable workload

1. Attach an isolated iOS Simulator to a same-tag macOS host.
2. Generate deterministic terminal output while exercising scroll, viewport resize, background/foreground, and surface remount transitions.
3. Record transport receipt, output application, render submission, Ghostty disposition, presentation, and replay reveal.
4. Fail if output advances without a corresponding current-generation presentation, or if any gate remains armed after its owner reports discard/failure/teardown.

Stop condition: The same workload completes with every admitted frame reaching a terminal disposition and newer work remaining deliverable.

## Candidate owner boundaries

- Mac producer and event queue: frame emission, subscriber lifetime, and backpressure.
- iOS output consumer: stream lifetime, chunk serialization, replay admission, and continuity.
- Viewport scheduler: generation changes, stale acknowledgements, and resize cancellation.
- Terminal surface: mount ownership, scroll generations, render submission, and teardown.
- Ghostty renderer: submitted-frame disposition and Metal presentation callback.
- Visibility lifecycle: foreground/background and verified-replay snapshot reveal.

## Existing evidence

- https://github.com/manaflow-ai/cmux/pull/7098 added recovery for render pipeline stalls.
- https://github.com/manaflow-ai/cmux/pull/7666 fixed a surface teardown deadlock.
- https://github.com/manaflow-ai/cmux/pull/8106 made verified replay atomic until presentation.
- https://github.com/manaflow-ai/cmux/pull/9146 fixed a replay loop caused by missing history continuity and added latency tracing.
- https://github.com/manaflow-ai/cmux/pull/10125 identifies discarded tokened render submissions as an unresolved terminal-disposition gap.

Root cause, proof, residual risk, and final workload results will be appended after verification.
