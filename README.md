<!--
  BANNER: drop a 600x200 banner here before launch.
  drag an image into a GitHub issue comment to get a CDN URL, then:
  <p align="center"><img src="URL" alt="rule-drift" width="600"></p>
-->

<p align="center">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-black">
</p>

# rule-drift

A Claude Code skill that checks whether the facts you cite are still true at the source, and tells you which of your docs to fix when they are not.

Tools like changedetection.io tell you a page changed. rule-drift tells you what the value changed to, and which of your files to fix.

## The Problem

Your content quotes numbers from other people's websites: a government limit, a quota, a price, a deadline. Those numbers change, and nobody tells you. The source page updates, your article does not, and you find out when a reader does. Tools that watch pages can say "this page changed," but they cannot tell you the number went from 60 to 80, or which of your files now needs fixing.

## Demo

<!-- TODO before launch: record a 3-to-10s GIF of a run that catches a DRIFT and names the affected doc.
     macOS screen-record, convert to GIF, drag into an issue to get the CDN URL, embed here.
     This is the single highest-ROI element in the README. -->

`[ demo GIF goes here ]`

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

You give it a list. Each item is one fact: the web page it lives on, a plain-English note for what to pull ("find the latest version number"), and which of your files uses it.

When it runs, it opens each page, reads out that one value, and compares it to what it saw last time. Each fact comes back as one of four things: OK (unchanged), DRIFT (changed), NEW (no record yet), or ERROR (could not read the page). You get a dated report, and for anything that changed, it names the file to fix.

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

**It tells you, it does not touch your files.** It flags what changed and which file to fix, then stops. If I let it rewrite a live page off a single number it read, one misread would quietly break a real page. That is worse than just making the edit yourself.

**It points at the exact file to fix.** It does not only say "a number changed." It says which of your files used that number. "Something changed" sends you hunting. "This changed, fix this file" is something you can just do.

**It runs free on Claude Code.** It uses the Claude Code plan you already have, so every check costs nothing and anyone with Claude Code can run it. The catch: it reads pages the simple way, so pages that need a full browser (heavy JavaScript, logins) will not work, and it says so plainly. Most pages are fine.

**The trick is not the check, it is everything around it.** Honestly, underneath this is just "go read a page," something you could do by hand. What makes it useful is the rest: a fixed list so you never forget a number, a saved history so it can spot changes, a link to the file to fix, and a schedule so it runs without you. It turns "remember to check this" into something that checks itself.

## What It Does Not Do

- It does not auto-edit your content. Drift is reported; you make the edit.
- It does not handle pages behind a login, or pages that need heavy JavaScript to load. If a page will not load as plain text, that target reports ERROR and needs a real browser, which is a separate path.
- It does not commit anything. It writes a report and the baseline files.

## License

[MIT](LICENSE)
