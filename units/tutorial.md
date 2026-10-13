---
objective: One story, in chapters, that takes a reader through the whole toolkit — from dotfiles to deployment
status: placeholder
---

The last unit of this series. Each tool has a README and a skill; nothing shows them working
together on one piece of work. This unit writes that: a tutorial told as a story, each chapter a
step of one workflow, every command in it run for real before it is written down.

Depends on: everything on the roadmap before it. Written last, against what was built.

## The shape (to be planned)

- **The people.** A small team, each with an agent that does part of the work: a developer, a
  devops engineer, an admin. Each has a role in the workflow and a reason to use a different
  part of the toolkit.
- **The work.** A basic todo app that exists as a repository. The team takes a new product idea
  for it from idea, through development, to a deployment.
- **Chapters**, each ending with something that runs. A first cut, to be revised when planned:

  | # | Chapter | Who | Tools |
  | --- | --- | --- | --- |
  | 1 | A new machine: install, onboard, a profile for the host | developer | `install.sh`, `anfs onboard`, `anfs profile` |
  | 2 | Dotfiles and making them yours: the user source, claiming a file | developer | `ppm`, `anfs mv`, `anfs owner` |
  | 3 | The team's source and the app's space: clone the todo app into it | admin | `anfs src`, `wsm` |
  | 4 | Services the app needs, on the laptop | developer | `pcm` |
  | 5 | Skills: what the agents know about this team's way of working | developer and agent | `psm` |
  | 6 | The idea, as a job: a branch in every repo it touches | developer and agent | `anfs work` |
  | 7 | An image for the app, and a machine from it | devops | `pmm image`, `pmm run`, provisioners |
  | 8 | The app as a container image | devops | `pcm image` |
  | 9 | The dashboard, and what the agents drive through the API | everyone | the toolkit API, anfs-ui |
  | 10 | Deploying: a target, a template, a deployment | devops and admin | the deploy tool |
  | 11 | A machine with no OS: the same image on hardware | admin | metal |
  | 12 | A new teammate joins: one command, and they are at chapter 6 | everyone | all of it |

- **Where it lives.** Public, in `anfs-io/docs` (or the website, built from it): to be decided.
- **How it is kept true.** Each chapter's commands are a script that runs in a box, so a change
  to a tool that breaks the tutorial fails a test.

`units/machine-images/README.md` is the first piece written this way (images and provisioners,
for one reader) and is the model for a chapter's level of detail.

## Plans

None yet. Planned when the deploy unit is built.
