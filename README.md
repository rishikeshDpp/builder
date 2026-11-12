# builder

builder skills: a small collection of Claude Code skills.

## Install

```
claude plugin marketplace add rishikeshDpp/builder
claude plugin install builder@builder
```

## Skills

- `/merge`: confirm PRs, watch required CI, fix failures, merge.

## Setup

Copy `skills/merge/config.example.json` to `~/.config/builder/config.json` and list your repos. Requires an authenticated `gh` CLI.
