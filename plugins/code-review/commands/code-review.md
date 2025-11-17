---
allowed-tools: Bash(glab issue view:*), Bash(glab mr view:*), Bash(glab issue list:*), Bash(glab mr list:*), Bash(glab mr view:*), Bash(glab mr diff:*)
description: Code review a merge request
disable-model-invocation: false
---

Provide a code review for the given merge request.

To do this, follow these steps precisely:

1. Use a Haiku agent to check if the merge request (a) is closed, (b) is a draft, (c) does not need a code review (eg. because it is an automated merge request, or is very simple and obviously ok), or (d) already has a code review from you from earlier. If so, do not proceed.
   ```bash
   glab mr view $MR_NUMBER --repo $REPO
   ```

2. Use another Haiku agent to give you a list of file paths to (but not the contents of) any relevant CLAUDE.md files from the codebase: the root CLAUDE.md file (if one exists), as well as any CLAUDE.md files in the directories whose files the merge request modified
   ```bash
   glab mr diff $MR_NUMBER --name-only --repo $REPO
   ```

3. Use a Haiku agent to view the merge request, and ask the agent to return a summary of the change
   ```bash
   glab mr view $MR_NUMBER --repo $REPO
   ```

4. Then, launch 5 parallel Sonnet agents to independently code review the change. The agents should do the following, then return a list of issues and the reason each issue was flagged (eg. CLAUDE.md adherence, bug, historical git context, etc.):
   a. Agent #1: Audit the changes to make sure they comply with the CLAUDE.md. Note that CLAUDE.md is guidance for Claude as it writes code, so not all instructions will be applicable during code review.
      ```bash
      glab mr diff $MR_NUMBER --repo $REPO
      ```
   b. Agent #2: Read the file changes in the merge request, then do a shallow scan for obvious bugs. Avoid reading extra context beyond the changes, focusing just on the changes themselves. Focus on large bugs, and avoid small issues and nitpicks. Ignore likely false positives.
      ```bash
      glab mr diff $MR_NUMBER --repo $REPO
      ```
   c. Agent #3: Read the git blame and history of the code modified, to identify any bugs in light of that historical context
      ```bash
      git blame -L <line>,<line> <file>  # 在repo中执行
      git log -p -S "keyword" -- <file>  # 在repo中执行
      ```
   d. Agent #4: Read previous merge requests that touched these files, and check for any comments on those merge requests that may also apply to the current merge request.
      ```bash
      glab mr list --repo $REPO --state merged --author @me
      ```
   e. Agent #5: Read code comments in the modified files, and make sure the changes in the merge request comply with any guidance in the comments.
      ```bash
      glab mr diff $MR_NUMBER --repo $REPO
      ```

5. For each issue found in #4, launch a parallel Haiku agent that takes the MR, issue description, and list of CLAUDE.md files (from step 2), and returns a score to indicate the agent's level of confidence for whether the issue is real or false positive. To do that, the agent should score each issue on a scale from 0-100, indicating its level of confidence. For issues that were flagged due to CLAUDE.md instructions, the agent should double check that the CLAUDE.md actually calls out that issue specifically. The scale is (give this rubric to the agent verbatim):
   a. 0: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
   b. 25: Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
   c. 50: Moderately confident. The agent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the MR, it's not very important.
   d. 75: Highly confident. The agent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach in the MR is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant CLAUDE.md.
   e. 100: Absolutely certain. The agent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.

6. Filter out any issues with a score less than 80. If there are no issues that meet this criteria, do not proceed.

7. Use a Haiku agent to repeat the eligibility check from #1, to make sure that the merge request is still eligible for code review.
   ```bash
   glab mr view $MR_NUMBER --repo $REPO
   ```

8. Finally, use the glab bash command to comment back on the merge request with the result.
   ```bash
   glab mr comment $MR_NUMBER --body "..." --repo $REPO
   ```

When writing your comment, keep in mind to:
   a. Keep your output brief
   b. Avoid emojis
   c. Link and cite relevant code, files, and URLs

**GitLab链接格式示例：**
```
https://gitlab.com/group/repo/-/blob/<commit-sha>/<file-path>#L10-L15
```

Examples of false positives, for steps 4 and 5:

- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (eg. missing or incorrect imports, type errors, broken tests, formatting issues, pedantic style issues like newlines). No need to run these build steps yourself -- it is safe to assume that they will be run separately as part of CI.
- General code quality issues (eg. lack of test coverage, general security issues, poor documentation), unless explicitly required in CLAUDE.md
- Issues that are called out in CLAUDE.md, but explicitly silenced in the code (eg. due to a lint ignore comment)
- Changes in functionality that are likely intentional or are directly related to the broader change
- Real issues, but on lines that the user did not modify in their merge request

Notes:

- Do not check build signal or attempt to build or typecheck the app. These will run separately, and are not relevant to your code review.
- Use `glab` to interact with GitLab (eg. to fetch a merge request, or to create inline comments), rather than web fetch
- Use `git` commands within the repo context for blame/history
- Make a todo list first
- You must cite and link each bug (eg. if referring to a CLAUDE.md, you must link it)
- For your final comment, follow the following format precisely (assuming for this example that you found 3 issues):

---

### Code review

Found 3 issues:

1. <brief description of bug> (CLAUDE.md says "<...>")

https://gitlab.com/group/project/-/blob/1d54823877c4de72b2316a64032a54afc404e619/README.md#L13-L17

2. <brief description of bug> (some/other/CLAUDE.md says "<...>")

https://gitlab.com/group/project/-/blob/1d54823877c4de72b2316a64032a54afc404e619/src/utils.js#L45-L50

3. <brief description of bug> (bug due to <file and code snippet>)

https://gitlab.com/group/project/-/blob/1d54823877c4de72b2316a64032a54afc404e619/src/main.js#L100-L105

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>

---

- Or, if you found no issues:

---

### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)
