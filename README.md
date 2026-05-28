<!--
  BANNER: drop a 600x200 banner here before launch.
  drag an image into a GitHub issue comment to get a CDN URL, then:
  <p align="center"><img src="URL" alt="rule-drift" width="600"></p>
-->

<p align="center">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-black">
</p>

# rule-drift

A Claude Code skill that catches when a fact you cite has changed at the source, shows you what is now out of date, and fixes every file once you approve.

Most page monitors stop at "this page changed." rule-drift reads the actual value, finds every file that still has the old one, and updates them after you sign off.

## The Problem

Your content quotes numbers from other people's websites: a government limit, a quota, a price, a deadline. Those numbers change, and nobody tells you. The source page updates, your article does not, and you find out when a reader does. Tools that watch pages can say "this page changed," but they cannot tell you the number went from 60 to 80, or which of your files now needs fixing.

## Demo

A run that catches a change looks like this:

```text
# rule-drift snapshot, 2026-05-28
| Target                   | Status | Current | Baseline |
| python-latest-stable     | OK     | 3.14.5  | 3.14.5   |
| node-latest-stable       | DRIFT  | 26.1.0  | 24.9.0   |
| kubernetes-latest-stable | OK     | 1.36.1  | 1.36.1   |

DRIFT  node-latest-stable  24.9.0 -> 26.1.0
Found "24.9.0" in 2 files:
  docs/install.md:12
  README.md:40
Update both to "26.1.0"? [y/n]
```

## Quick Start

```bash
# Install as a Claude Code skill
git clone https://github.com/hwajpark/rule-drift ~/.claude/skills/rule-drift

# Point it at your sources
cp ~/.claude/skills/rule-drift/examples/rules.config.example.json ./rules.config.json
# edit rules.config.json: add your targets

# Capture a baseline (first run)
/rule-drift --update-baseline

# Later, check for drift
/rule-drift
```

## How It Works

You give it a list. Each item is one fact: the web page it lives on, a plain-English note for what to pull ("find the latest version number"), and where you cite it.

When it runs, it opens each page and compares the value to what it saw last time. If something changed, it shows you the old value, the new value, and every file in your project that still has the old one. You check that list, and if it looks right, it updates all of them at once. Nothing changes until you say yes.

Under the hood it checks all the pages at once, so a long list still finishes fast.

## Configuration

One entry per fact. The included [`examples/rules.config.example.json`](examples/rules.config.example.json) tracks the latest stable versions of Python, Node.js, and Kubernetes, so it runs out of the box.

```json
{
  "targets": [
    {
      "id": "python-latest-stable",
      "name": "Python latest stable release",
      "url": "https://en.wikipedia.org/wiki/Python_(programming_language)",
      "extract": "From the infobox, return the latest stable release version of Python. Return only the version string.",
      "doc_slug": "docs/requirements",
      "notes": "Our setup docs pin a minimum Python version; bump them when a new stable lands."
    }
  ]
}
```

Want a real-world example? [`examples/advanced-korea-gov.config.json`](examples/advanced-korea-gov.config.json) tracks Korean government pages. Some of those need a full browser, so it also shows how the skill handles pages it cannot read.

## Design Decisions

**Plain words, not code, to find the value.** I tell it what to look for in plain English, like "find the latest version number." The other option is to point at an exact spot in the page's code. That is faster, but it breaks the moment someone redesigns the page, even when the number is still sitting right there. Plain words keep working through a redesign. Worth the small extra cost.

**It remembers the last value, so it can spot a change.** To know a number changed, you have to know what it was before. The first run writes down each value and where it found it. Every run after that compares now against last time. No memory, no way to catch a change.

**It does the fixing, you do the deciding.** The point is not to hand you a to-do list, it is to do the boring work. When a value changes, it finds every file that still has the old one, prepares the edits, and applies them once you approve. It does the finding and the typing; you make the call.

**It always shows the plan before it touches anything.** Nothing changes until you have seen exactly what will change. A number like "60" can mean five different things across your files, and you are the one who knows which should actually update. So it shows you every match first and lets you drop the ones that should stay.

**It runs free on Claude Code.** It uses the Claude Code plan you already have, so every check costs nothing and anyone with Claude Code can run it. The catch: it reads pages the simple way, so pages that need a full browser (heavy JavaScript, logins) will not work, and it says so plainly. Most pages are fine.

**The trick is not the check, it is everything around it.** Honestly, underneath this is just "go read a page," something you could do by hand. What makes it useful is the rest: a fixed list so you never forget a number, a saved history so it can spot changes, the ability to fix every file at once, and a schedule so it runs without you. It turns "remember to check this and fix it everywhere" into a single yes.

## What It Does Not Do

- It does not change anything without your approval. You see every edit before it touches a file, and you can drop any of them.
- It does not handle pages behind a login, or pages that need heavy JavaScript to load. If a page will not load as plain text, that target reports ERROR and needs a real browser, which is a separate path.
- It does not commit to git. It edits the files and updates the baseline; committing is yours.

## License

[MIT](LICENSE)
