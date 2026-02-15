# Email Triage Skill

A [Claude Skill](https://docs.claude.com) that scans your Gmail inbox, classifies every message using your own rules, applies Gmail labels, and archives suspected junk — all in one pass.

## What it does

1. Reads your `triage-rules.md` file to learn what matters and what doesn't
2. Scans your Gmail inbox via the Gmail MCP connector
3. Classifies each email as **Suspected Junk**, **Action Needed**, **Informational**, or **Unknown**
4. Applies the corresponding Gmail labels via Chrome browser automation
5. Archives suspected junk and logs decisions for review

## Requirements

- **Gmail MCP connector** — for reading emails (read-only)
- **Chrome browser automation** — for applying labels and archiving in Gmail's web UI
- **`triage-rules.md`** — your custom rules file in the working folder (a template is provided in `references/`)

## Installation

Install the `.skill` package through Claude's Settings UI, or copy the skill directory into your skills folder.

To create a `.skill` package from source:

```bash
python3 /path/to/skill-creator/scripts/package_skill.py email-triage
```

## Usage

Tell Claude to triage your email:

> "Triage my email"

The skill supports two modes (configured in your `triage-rules.md`):

- **Review mode** — presents a summary table with proposed classifications and waits for your approval before labeling
- **Auto mode** — classifies and labels automatically, then shows a summary of what was done

## Customizing rules

Copy `references/triage-rules-template.md` to your working folder as `triage-rules.md` and edit it to define your own rules, trusted domains, and trusted sources. The skill will adapt to whatever rules you provide.

## Project structure

```
email-triage/
├── SKILL.md                        # Main skill instructions
├── README.md                       # This file
├── .gitignore
├── references/
│   └── triage-rules-template.md    # Blank template for users
└── evals/
    ├── evals.json                  # 15 automated test cases
    └── files/                      # Test fixtures
        ├── triage-rules-review.md
        ├── triage-rules-auto.md
        └── existing-log.md
```

## Running evals

The `evals/` directory contains 15 test cases covering classification logic, rule parsing, logging, mode handling, and edge cases. Run them using the skill-creator framework:

```bash
# Validate the eval file
python3 /path/to/skill-creator/scripts/validate_json.py evals/evals.json

# Prepare a specific eval workspace (0-indexed)
python3 /path/to/skill-creator/scripts/prepare_eval.py email-triage 0 --output-dir workspace/eval-1/with_skill
```

## License

MIT
