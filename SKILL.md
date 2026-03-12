---
name: fix-bot-reviews
description: >
  Fetch and fix bot review comments on a GitHub PR from Cursor Bugbot, Sentry, Copilot,
  Gemini Code Assist, and other CI bots. Use when the user says "fix bot reviews",
  "fix pr comments", "address bot reviews", "fix review comments", "handle pr feedback",
  "resolve pr comments", "resolve bot comments", or wants to process automated review
  feedback on a pull request. Accepts optional PR URL or number as argument; defaults to
  the PR for the current git branch.
---

# Review PR Bot Comments

Address automated review comments from CI bots on a GitHub PR.

**Supported bots:** Cursor Bugbot, GitHub Copilot, Gemini Code Assist, Sentry, Linear, GitHub Actions

## Usage

```
/fix-bot-reviews                    # PR for current branch
/fix-bot-reviews 1234               # PR #1234
/fix-bot-reviews https://github.com/owner/repo/pull/1234
```

## Workflow

### Phase 1: Fetch Comments

1. Parse arguments to extract PR number. If a full URL is provided, extract `owner/repo` and PR number from it.
2. If no PR specified, detect from current branch via `gh pr view --json number`.
3. Run the fetch script:
   ```bash
   bash ~/.claude/skills/fix-bot-reviews/scripts/fetch_reviews.sh [--pr NUMBER] [--repo OWNER/REPO]
   ```
4. Parse the JSON output. Each comment has: `id`, `bot`, `path`, `line`, `body`, `url`, `diff_hunk`, `in_reply_to_id`.
5. Filter out reply comments (`in_reply_to_id != null`) — these are threaded replies, not top-level reviews.
6. If zero comments found, report "No bot review comments found" and stop.

### Phase 2: Present Review Table

Display all bot comments in a markdown table:

```
| #  | Bot            | File                  | Line | Comment (truncated to 80 chars)       |
|----|----------------|-----------------------|------|---------------------------------------|
| 1  | copilot[bot]   | src/api/handler.ts    | 42   | Consider adding error handling for... |
| 2  | cursor-bugbot  | src/utils/parse.ts    | 15   | Potential null pointer dereference...  |
```

After the table, ask the user:
> "I found **N** bot review comments. Let's go through each one. For each comment, I'll show the full details and you can choose to **fix** or **skip** it."

### Phase 3: Address Each Comment

For each comment, sequentially:

1. Show the full comment details:
   - Bot name, file path, line number, full URL
   - The diff hunk for context (if available)
   - The full comment body
2. Read the relevant file and surrounding code context
3. Propose a fix based on the bot's suggestion
4. Ask the user: **"Fix this? (fix/skip)"**
5. Based on response:
   - **fix**: Apply the code change. Record status as `FIXED`.
   - **skip**: Move on. Record status as `SKIPPED`.
6. Track each comment's resolution in an internal list: `[{id, bot, path, line, status, summary}]`

### Phase 4: Commit & Push

After all comments are addressed:

1. Show a summary of changes:
   ```
   Fixed: N comments | Skipped: M comments
   ```
2. If any fixes were made, ask: **"Ready to commit and push these fixes?"**
3. On approval:
   - Stage only the modified files related to fixes
   - Create a commit with message: `fix: address bot review comments on PR #NUMBER`
   - Push to the current branch
4. If no fixes were made, skip to Phase 5.

### Phase 5: Reply to PR Comments

After push is confirmed, for each comment in the tracked list:

- **FIXED comments**: Reply on the PR and resolve the thread
  ```bash
  # Reply to inline review comment
  gh api "repos/OWNER/REPO/pulls/PR/comments/COMMENT_ID/replies" \
    -f body="Fixed in $(git rev-parse --short HEAD). Thank you for the review."

  # For PR-level comments, reply as issue comment mentioning the original
  gh pr comment PR --repo OWNER/REPO --body "Addressed: SUMMARY. Fixed in $(git rev-parse --short HEAD)."
  ```

- **SKIPPED comments**: Reply acknowledging the review
  ```bash
  gh api "repos/OWNER/REPO/pulls/PR/comments/COMMENT_ID/replies" \
    -f body="Acknowledged — skipping this suggestion as it's not applicable. Thank you for the review."
  ```

**Important**: For inline review comments (those with a `path`), use the replies API endpoint. For PR-level or issue comments, use `gh pr comment`.

### Phase 6: Final Report

Display the full report table:

```
## Review Report — PR #NUMBER

| #  | Bot            | File               | Line | Status  | Action Summary                    |
|----|----------------|--------------------|------|---------|-----------------------------------|
| 1  | copilot[bot]   | src/api/handler.ts | 42   | FIXED   | Added try-catch error handling    |
| 2  | cursor-bugbot  | src/utils/parse.ts | 15   | SKIPPED | Not applicable — already guarded  |

**Total: N comments | Fixed: X | Skipped: Y**
**Commit: abc1234**
**All comments have been replied to on the PR.**
```

## Error Handling

- If `gh` CLI is not authenticated, prompt user to run `gh auth login`.
- If PR has no bot comments, report cleanly and exit.
- If push fails (e.g., branch protection), inform user and skip Phase 5.
- If a reply to a comment fails, log the error but continue with remaining comments.

## Customization

To add more bots, edit the `BOT_LOGINS` array in `scripts/fetch_reviews.sh`.
