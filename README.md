# polkit AI Skills

Skills and environments for AI-powered polkit issue triage and reproduction.

## Structure

- `GEMINI.md` — Agent instructions for Gemini CLI
- `skills/` — Domain knowledge organized by component
- `environments/` — Dockerfiles with systemd + polkit + Gemini CLI
- `.github/workflows/build-images.yml` — Builds and pushes container images

## How it works

When a maintainer comments `@ai-triage` on a polkit issue:

1. Gemini REST API triages the issue (assess, label, ask for info)
2. A Docker container is started with systemd + polkit running
3. This repo is cloned into the container
4. Gemini CLI runs inside, iterating on a reproducer until it works
5. The reproducer is posted to the issue
6. If the agent learned something new, it opens a PR to add a skill file

## Skills

Skills are markdown files containing domain expertise. They grow over time
as the agent handles more issues. Each skill documents a technique for
reproducing a specific class of polkit bug.

## Container images

Pre-built images are available at:
- `ghcr.io/vmihalko/polkit-ai-skills:fedora`
- `ghcr.io/vmihalko/polkit-ai-skills:ubuntu`
