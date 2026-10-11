---
status: complete
summary: ai/herdr installs via brew on every platform; Linux arm64 gets a prebuilt bottle
started_at: 2026-10-07
completed_at: 2026-10-07
deviations: No box snapshot taken; the bottle installs in 8s, so one isn't needed. The protocol check moved to plan 04, because jq is not in a bare box.
---

- `package.yml`: `brew: {macos: [herdr]}` changed to `brew: [herdr]`, and version bumped to 0.1.1.
- Debian 13 box (from `debian-bootstrap`): `ppm install ai/herdr` poured `herdr--0.9.3.arm64_linux.bottle.tar.gz` in **8s**. No source build. The local `brew info --json` on macOS only lists the host's own bottle, which made it look like Linux had none.
- `herdr server` starts headless in the box, and `herdr session list` shows `default running`.
- **Version drift:** brew now ships herdr **0.9.3**. Firstmate (ref 42dd906) is verified up to 0.8.0, and the macOS host still has 0.8.2, because ppm doesn't upgrade a formula that's already installed. Unit `factory-e2e` is where drift is caught.
