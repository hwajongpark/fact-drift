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

Unlike page-diff monitors such as changedetection.io, rule-drift extracts the actual value and names the doc you need to fix.

## The Problem

If your content quotes numbers that live on someone else's website (a government threshold, a published quota, a price, a policy date), those numbers go stale silently. The source page changes, your content does not, and you find out when a reader does. Generic page-change monitors tell you "this page changed." They do not tell you the value went from 60 to 80, and they have no idea which of your files now needs an edit.

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

`rule-drift` reads a JSON config of targets. Each target is one fact you care about: a URL, a natural-language instruction describing the value to extract, and the slug of the doc that cites it. On a run, it fans out one Claude subagent per target in parallel, each fetching its page and returning a single extracted value with a one-line provenance note. It diffs every value against a stored baseline of `{ value, source_url, captured_at }` and classifies each as OK, DRIFT, NEW, or ERROR. The output is a dated snapshot report; on a DRIFT it names the exact doc slug that now needs an edit.

The extraction is natural-language, not a CSS selector. You write "find the minimum points required to qualify," not `div.table > tr:nth-child(3)`. So a target keeps working when the source page is restyled, which is the failure mode that breaks selector-based monitors.

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

For a real-world example against JavaScript-rendered government pages, and how the skill reports the ones a plain fetch cannot read, see [`examples/advanced-korea-gov.config.json`](examples/advanced-korea-gov.config.json).

## Design Decisions

**Plain words, not code, to find the value.** I tell it what to look for in plain English, like "find the latest version number." The other option is to point at an exact spot in the page's code. That is faster, but it breaks the moment someone redesigns the page, even when the number is still sitting right there. Plain words keep working through a redesign. Worth the small extra cost.

**It remembers the last value, so it can spot a change.** To know a number changed, you have to know what it was before. The first run writes down each value and where it found it. Every run after that compares now against last time. No memory, no way to catch a change.

**It tells you, it does not touch your files.** It flags what changed and which file to fix, then stops. If I let it rewrite a live page off a single number it read, one misread would quietly break a real page. That is worse than just making the edit yourself.

**It points at the exact file to fix.** It does not only say "a number changed." It says which of your files used that number. "Something changed" sends you hunting. "This changed, fix this file" is something you can just do.

**It runs free on Claude Code.** It uses the Claude Code plan you already have, so every check costs nothing and anyone with Claude Code can run it. The catch: it reads pages the simple way, so pages that need a full browser (heavy JavaScript, logins) will not work, and it says so plainly. Most pages are fine.

**The trick is not the check, it is everything around it.** Honestly, underneath this is just "go read a page," something you could do by hand. What makes it useful is the rest: a fixed list so you never forget a number, a saved history so it can spot changes, a link to the file to fix, and a schedule so it runs without you. It turns "remember to check this" into something that checks itself.

## What It Does Not Do

- It does not auto-edit your content. Drift is reported; you make the edit.
- It does not handle login-walled or JavaScript-rendered pages. If a fetch returns boilerplate, the target reports ERROR and needs a real browser, which is a separate path.
- It does not commit anything. It writes a report and the baseline files.

## License

[MIT](LICENSE)
