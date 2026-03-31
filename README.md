# LinkedIn Easy Apply - Claude Code Skill

A [Claude Code](https://claude.ai/claude-code) skill that automates LinkedIn Easy Apply job applications using the [Playwright MCP](https://github.com/anthropics/mcp-playwright) browser.

Point it at your resume, tell it what roles to search for, and it will search LinkedIn, filter out staffing agencies and scam postings, fill out application forms using only verified resume data, and submit up to 20 applications per session.

## Demo

[![LinkedIn Easy Apply Demo](https://img.youtube.com/vi/SVP6IP11TQA/maxresdefault.jpg)](https://youtu.be/SVP6IP11TQA)

## Features

- Searches LinkedIn Jobs with configurable filters (remote, Easy Apply, recency)
- Filters out staffing agencies, fake postings, and low-quality listings automatically
- Answers application questions strictly from your resume (never fabricates experience)
- Bails out of applications where you're underqualified (3+ "No"/"0" answers)
- Unchecks "Follow company" on every application
- Handles multiple ATS systems (LinkedIn native, Workable, Lever)
- Anti-detection with randomized delays between actions
- Context-efficient Playwright MCP usage (minimizes snapshot bloat)

## Prerequisites

- [Claude Code](https://claude.ai/claude-code) CLI or desktop app
- [Playwright MCP server](https://github.com/anthropics/mcp-playwright) configured in `~/.claude.json`
- A LinkedIn account logged in via the Playwright browser
- Your resume uploaded to LinkedIn and available as a local file (markdown or text)

## Installation

Copy `SKILL.md` into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/linkedin-easy-apply
cp SKILL.md ~/.claude/skills/linkedin-easy-apply/
```

## Configuration

Edit the `Customization` section at the bottom of `SKILL.md` and replace placeholders with your info:

- `<RESUME_PATH>` - Path to your resume file
- `<Your City>` - Your city for location fields
- `<Your University>` - For education dropdowns
- `<Your Degree>` - For education dropdowns
- Salary thresholds and target compensation
- Search keywords for your target roles

### Playwright MCP Config

Add this to your `~/.claude.json` MCP server config:

```json
{
  "mcpServers": {
    "playwright-mcp": {
      "command": "npx",
      "args": [
        "@anthropic-ai/mcp-playwright",
        "--snapshot-mode", "none",
        "--block-service-workers",
        "--blocked-origins", "chrome-extension://invalid/",
        "--output-mode", "file",
        "--no-sandbox"
      ]
    }
  }
}
```

## Usage

In Claude Code:

```
/linkedin-easy-apply senior AI engineer python remote
```

Or without arguments to be prompted for search keywords:

```
/linkedin-easy-apply
```

## How It Works

1. Searches LinkedIn with your keywords, filtered for Remote + Easy Apply + past 30 days
2. For each result, visits the job page and runs automated checks (staffing agency, salary, already applied, qualification match)
3. Clicks Easy Apply and fills out the multi-step form using your resume data
4. Unchecks "Follow company", submits, and moves to the next job
5. Reports a summary table of all applied and skipped jobs

## Disclaimer

This tool is provided for educational and personal productivity purposes only. Use it at your own risk. Automated interaction with LinkedIn may violate their [Terms of Service](https://www.linkedin.com/legal/user-agreement). The authors are not responsible for any account restrictions, bans, or other consequences resulting from use of this tool. Be aware that LinkedIn actively detects and throttles automated activity.

This tool does not store or transmit your credentials. All browser sessions run locally via Playwright MCP.

## License

MIT
