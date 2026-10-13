---
objective: An unbiased read of the whole toolkit by agents who have seen none of our reasoning — what is good, what is confusing, what should be cut
status: placeholder
---

Everyone who has judged anfs so far helped build it, the agents included: each one read the
plans and the CLAUDE.md before it read the code, so each one judged the work by its own
intentions. This unit asks people who were not in the room.

Run before `tutorial`, so the tutorial is written about what survived. A smaller first pass
(lenses 1 and 3 only) is worth running as soon as `machine-images` is merged: a finding that
says "cut this" is cheapest before deploy and metal are built on it.

## Rules that keep it honest

- **A clean agent.** A new session with no memory of this work, no access to this docs repo,
  and no CLAUDE.md beyond what a stranger cloning the public repos gets. It is given a task and
  a box, never our description of what anfs is for.
- **It is asked to find what is wrong.** The brief says a report with no serious findings will
  be read as a failed audit, and that "this should not exist" is an acceptable conclusion.
- **Claims are checked by doing.** A finding names the command run and what it printed. An
  opinion with no run behind it is marked as one.
- **The auditor does not fix anything**, and no one who built a part answers for it in the
  report. Our replies go in a separate file, after.
- **Different lenses are different agents**, blind to each other, so one's framing does not
  become all of theirs.

## Lenses (to be planned)

| # | Lens | The agent is | Asked |
| --- | --- | --- | --- |
| 1 | First hour | A developer with a new Mac, and another with a Debian box, given only the public README | Install, set up dotfiles, run a service, build an image. Where did you stop, guess, or read source to continue? |
| 2 | Second machine | The same person a week later | Make a second machine match the first. Break something and recover |
| 3 | Why not what exists | A sceptic who knows chezmoi, Nix and home-manager, Ansible, devcontainers, Packer, Lima, Terraform | For each tool: what does it do that the established one does not, and what does the established one do that it cannot? Which tools should be a thin layer over something else? |
| 4 | Architecture | A senior reviewer reading the code cold | Six tools in bash: where does that cost more than it saves? What is duplicated, what is one bug away from data loss, what will not survive a tenth contributor? |
| 5 | Safety | A security reviewer | What runs as root, what is fetched and trusted, what does an untrusted source get to execute, what does implode leave or take |
| 6 | Agent as user | An agent given a task and only `--help`, `--json` and the skills | Can it do the task without reading source? Where are the skills wrong or missing? |
| 7 | Contributor | A stranger with a fix to send | Clone `dev-anfs`, run the tests, change something, prove it. How long, and what was unexplained? |

## What comes out

- One report per lens, findings ranked by how much they would matter to a stranger.
- A synthesis by one more clean agent: what several lenses found independently, what to cut,
  what to rename, what to build before anything else. Disagreements between lenses are kept,
  not averaged.
- Our answer to each finding: fix (a plan), decline (why), or not now.

## Plans

None yet.
