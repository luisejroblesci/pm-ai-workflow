---
name: address-pr-reviews
description: Address PR review comments and feedback automatically
parameters:
  - name: pr_number
    description: Specific PR number to address (optional, if not provided will check all open PRs)
    required: false
---

# Address PR Reviews

You are a PR review response specialist. Your task is to address all review comments and feedback on GitHub Pull Requests by making the necessary code changes.

## Input Parameters
- PR Number: {{pr_number}} (if empty, process all open PRs authored by @me)

## Your Task Workflow

### 1. Get PR Details
{{#if pr_number}}
- Use `gh pr view {{pr_number}} --json number,title,headRefName,url,reviewDecision` to get PR details
- Get the branch name from the output
{{else}}
- Use `gh search prs --author=@me --state=open --json number,title,url,repository --limit 20` to list all open PRs
- If multiple PRs are found, show the list to the user and ask which one(s) they want to address
- For the selected PR(s), run `gh pr view <PR_NUMBER> --json number,title,headRefName,url,reviewDecision` to get the branch name and review decision
{{/if}}

### 2. Retrieve All Comments and Review Feedback
For each PR being processed:

**A. Get regular issue comments:**
```bash
gh pr view <PR_NUMBER> --json comments --jq '.comments[] | "\(.author.login) (\(.createdAt)):\n\(.body)\n---"'
```

**B. Get review threads (inline code comments) with their thread IDs:**
```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 10) {
              nodes {
                body
                path
                line
                diffHunk
                author { login }
                createdAt
              }
            }
          }
        }
      }
    }
  }
' -f owner=<OWNER> -f repo=<REPO> -F number=<PR_NUMBER>
```

Record each thread's `id` alongside its comments — you'll need these IDs to resolve threads after addressing them.

**C. Get review summaries:**
```bash
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews --jq '.[] | "\(.user.login) - \(.state) (\(.submitted_at)):\n\(.body)\n---"'
```

### 3. Analyze All Feedback
Carefully review all comments and identify:
- **Requested changes**: What specific changes are reviewers asking for?
- **Questions**: Are there any questions that need to be addressed in code or comments?
- **Suggestions**: Code improvement suggestions or alternative approaches
- **Blockers**: Critical issues that must be resolved (CHANGES_REQUESTED reviews)
- **File locations**: Note which files and line numbers need attention (from review comments)

Create a summary like:
```
📋 Review Feedback Summary:
1. [File: path/to/file.ts:45] @reviewer - Suggestion: Use const instead of let
2. [File: path/to/other.ts:102] @reviewer - Required: Add error handling
3. [General] @reviewer - Question: Why this approach vs. alternative?
```

### 4. Switch to PR Branch
```bash
git fetch origin
git checkout <BRANCH_NAME>
git pull origin <BRANCH_NAME>
```

### 5. Apply Fixes
For each piece of feedback:

**Code changes:**
- Use Read tool to examine the relevant files and surrounding context
- Use Edit tool to make precise changes that address the feedback
- For inline review comments, make sure to address the specific line/code mentioned
- Ensure changes align with the project's style and patterns

**Documentation/Comments:**
- If reviewers asked questions, add clarifying comments in the code
- Update docstrings or README if requested

**Tests:**
- Add or update tests if reviewers requested test coverage
- Ensure existing tests still pass

### 6. Verify Changes Locally
Before committing, verify:
- Run linter: `pnpm lint` (or project-specific lint command)
- Run formatter: `pnpm format` (or project-specific format command)
- Run tests: `pnpm test` (or project-specific test command)
- If any fail, fix issues before proceeding

### 7. Commit and Push Changes
```bash
git add -A
git commit -m "Address PR review feedback

- [Summarize key changes made]
- [Reference specific review comments if applicable]

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
git push origin <BRANCH_NAME>
```

### 8. Resolve Addressed Review Threads
After pushing, resolve each inline review thread that was fully addressed by calling the GitHub GraphQL mutation:

```bash
gh api graphql -f query='
  mutation($threadId: ID!) {
    resolveReviewThread(input: {threadId: $threadId}) {
      thread {
        id
        isResolved
      }
    }
  }
' -f threadId=<THREAD_ID>
```

- Run this for **every thread whose feedback was fully implemented**.
- Skip threads that are still unresolved, need clarification, or were intentionally left for the reviewer to re-evaluate.
- Only resolve threads for unresolved (`isResolved: false`) threads — skip any already resolved.

### 9. Respond to Comments (Optional but Recommended)
If appropriate, add a comment to acknowledge the changes:
```bash
gh pr comment <PR_NUMBER> --body "✅ Addressed review feedback:
- [Change 1]
- [Change 2]

Thanks for the review!"
```

## Important Guidelines

### Code Quality
- Make targeted changes that directly address feedback
- Don't over-engineer or add unnecessary changes
- Maintain consistency with existing code style
- Preserve functionality unless explicitly asked to change it

### Communication
- If a review comment is unclear, note it in your summary and ask the user for clarification
- If you disagree with a suggestion or can't implement it, explain why
- If feedback conflicts, ask the user to prioritize

### Safety
- Always verify changes locally before pushing
- Don't bypass review comments - address each one thoughtfully
- If changes are substantial, summarize what you did

### Repository Context
- Adapt commands to the specific project structure
- Use the project's specific lint/test/format commands
- Respect pre-commit hooks and CI/CD requirements

## Output Format

For each PR processed, provide:
```
📝 PR #<NUMBER>: <TITLE>
Branch: <BRANCH_NAME>

📋 Review Feedback Found:
- <Count> review comments
- <Count> inline code comments
- <Count> general comments

🔧 Changes Applied:
1. <Description of change 1>
2. <Description of change 2>
...

✔️ Resolved Threads: <Count> of <Total> threads marked as resolved

✅ Status: [COMPLETED | NEEDS CLARIFICATION | PARTIALLY ADDRESSED]

🚀 Pushed to: <BRANCH_NAME>
```

## Edge Cases

- **No comments found**: Inform user that PR has no review feedback to address
- **Already approved**: Inform user that PR is already approved (but still can address comments)
- **Changes requested but unclear**: List unclear feedback and ask user for guidance
- **Multiple PRs**: Process them one by one, or ask user which to prioritize
- **Merge conflicts**: Alert user and ask how to proceed
- **Thread resolution fails**: Log the failure but continue — resolving is best-effort and shouldn't block the overall workflow
- **Thread already resolved**: Skip it (check `isResolved: true` before calling the mutation)

Now execute this workflow for the specified PR(s).
