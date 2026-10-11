---
objective: Day-to-day factory work runs Firstmate on herdr inside a VM, attached from the host — not on the host and not in a container
status: pending
---

Outline only. Write the plans after `factory-e2e` is complete, since that unit will show what
the VM needs.

Depends on: `firstmate-foundation` and `factory-e2e` units complete.

## Decisions

- **VM over container for real work:**
  - Workers run with permissions bypassed, so a VM is a real boundary: its own kernel, no host `$HOME`, scoped credentials. The test containers run `privileged: true`, so they are not a boundary.
  - It has systemd and real login shells for the long-running herdr server and supervision.
  - The whole factory can be snapshotted and reset.
- **The host is only the captain's seat.** Attach with `herdr --remote <vm>`. Firstmate's remote secondmates require herdr, so adding VMs later means adding crews.

## Plans (outline)

1. `01-image`: pim `anfs-dev` image with `anfs: {packages: [firstmate]}`.
2. `02-services`: a systemd user unit for `herdr --session firstmate server` (replaces `firstmate up`'s background start in the VM).
3. `03-creds`: a scoped GitHub deploy token and the Claude token via fnox/1Password, so no host credentials.
4. `04-attach`: `firstmate remote <vm>` → `herdr --remote`.
5. `05-run-e2e`: run `factory-e2e` against the VM.

## Completion Criteria

- From the host, `firstmate remote factory` attaches to a running crew in the VM.
- `factory-e2e` Tier 1 passes against the VM.
- A VM snapshot/reset restores a clean factory in minutes.
