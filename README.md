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

**Natural-language extraction, not CSS selectors.** Targets describe the value in plain English ("the latest stable version"), not a DOM path. Selector-based monitors are cheaper but break every time a page is restyled, which is the most common way these checks silently rot. Reading the page with an LLM trades a little cost for resilience: the check survives a redesign as long as the value is still there.

**A stored baseline, because the point is change, not current value.** Anyone can read today's number. Detecting that it moved since last time needs memory, so each reading is saved as `{value, source_url, captured_at}` and every run compares against the prior truth.

**Report and stop, never auto-edit.** It flags drift and names the doc to fix; it never touches your content. Letting an LLM rewrite a live doc from a single extracted value fails worse than the problem it solves. A human makes the edit.

**Slug linkage, so drift is actionable.** Each target maps to the doc that cites it. "The version changed" is information; "the version changed, update `docs/requirements`" is a task.

**Runs on the Claude Code subscription, not an API runner.** Zero cost per run, usable by anyone with Claude Code. The tradeoff is the plain-fetch limit: JavaScript-rendered or login-walled pages error and need the browser escape hatch above. For server-rendered sources, the cheap path is enough.

**The value is the harness, not the check.** Underneath, this is a page-read you could do by hand. What makes it reliable is everything around it: a frozen target list so nothing is forgotten, a baseline so change is detectable, slug linkage so drift is actionable, and a schedule so it runs without you. It turns "remember to check the data" into a mechanism that cannot be forgotten.

## What It Does Not Do

- It does not auto-edit your content. Drift is reported; you make the edit.
- It does not handle login-walled or JavaScript-rendered pages. If a fetch returns boilerplate, the target reports ERROR and needs a real browser, which is a separate path.
- It does not commit anything. It writes a report and the baseline files.

## License

[MIT](LICENSE)
