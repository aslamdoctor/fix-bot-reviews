# fix-bot-reviews

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that fetches bot review comments from a GitHub PR, helps you fix or skip each one, and automatically replies on the PR marking them resolved.

## Supported Bots

- **Cursor Bugbot** — AI code review suggestions
- **GitHub Copilot** — Copilot code review
- **Gemini Code Assist** — Google's AI code review
- **Sentry** — Error tracking and code suggestions
- **Linear** — Issue tracker bot comments
- **GitHub Actions** — CI/CD bot comments

## Install

```bash
claude install aslamdoctor/fix-bot-reviews
```

## Usage

```
/fix-bot-reviews                    # PR for current branch
/fix-bot-reviews 1234               # PR #1234
/fix-bot-reviews https://github.com/owner/repo/pull/1234
```

Or just say things like:
- "fix pr comments"
- "address bot reviews"
- "resolve pr comments"

## How It Works

The skill runs through 6 phases:

### 1. Fetch Comments
Pulls all bot review comments from the PR using the GitHub API — inline code comments, PR-level reviews, and issue comments.

### 2. Present Review Table
Displays a summary table of all bot comments:

```
| #  | Bot            | File               | Line | Comment                               |
|----|----------------|--------------------|------|---------------------------------------|
| 1  | copilot[bot]   | src/api/handler.ts | 42   | Consider adding error handling for... |
| 2  | cursor-bugbot  | src/utils/parse.ts | 15   | Potential null pointer dereference...  |
```

### 3. Address Each Comment
Walks through each comment one-by-one:
- Shows full comment body, diff hunk, and code context
- Proposes a fix
- Asks you to **fix** or **skip**

### 4. Commit & Push
After all comments are addressed:
- Stages only the files that were modified
- Commits with a descriptive message
- Pushes to the branch (with your approval)

### 5. Reply on PR
Automatically replies to each comment on the PR:
- **Fixed** — "Fixed in `abc1234`. Thank you for the review."
- **Skipped** — "Acknowledged — skipping this suggestion as it's not applicable."

### 6. Final Report
Shows a complete summary:

```
| #  | Bot            | File               | Line | Status  | Action Summary                   |
|----|----------------|--------------------|------|---------|----------------------------------|
| 1  | copilot[bot]   | src/api/handler.ts | 42   | FIXED   | Added try-catch error handling   |
| 2  | cursor-bugbot  | src/utils/parse.ts | 15   | SKIPPED | Not applicable — already guarded |

Total: 2 comments | Fixed: 1 | Skipped: 1
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`) — authenticated via `gh auth login`
- `jq` — for JSON processing

## Customization

To add more bots, edit the `BOT_LOGINS` array in `scripts/fetch_reviews.sh`:

```bash
BOT_LOGINS=(
  "cursor-bugbot"
  "cursor-bugbot[bot]"
  "copilot[bot]"
  "github-copilot[bot]"
  "gemini-code-assist[bot]"
  "sentry-io[bot]"
  "linear[bot]"
  "your-custom-bot[bot]"  # add your bot here
)
```

## License

MIT
