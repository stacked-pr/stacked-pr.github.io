## The Problem

Code review has always been a bottleneck in software development. In the age of LLM-assisted coding, it's increasingly common for developers to create multiple PRs in a single day. The problem? Waiting for PR approval is counter-productive, especially when your next task depends on code that's still sitting in review.

You could just wait for approval before starting the next task, but that's inefficient. You could create a PR based on another unmerged PR, but then what happens when changes are requested? Enter stacked PRs: a workflow that lets you keep moving forward while maintaining clean, reviewable code.

## Level 0: Independent PRs (The Easy Mode)

The ideal scenario is when PRs are completely independent of each other. Think backend and frontend working in parallel: the backend team focuses on API implementation while the frontend team builds the UI based on an agreed-upon specification.

```bash
# Branch 1: English numbers
git checkout -b english-numbers
echo "print('one')" > english.py
echo "print('two')" >> english.py
echo "print('three')" >> english.py
git add .
git commit -m 'Add English number printer'

# Branch 2: Spanish numbers (independent)
git checkout main
git checkout -b spanish-numbers
echo "print('unos')" > spanish.py
echo "print('dos')" >> spanish.py
echo "print('tres')" >> spanish.py
git add .
git commit -m 'Add Spanish number printer'

# Branch 3: French numbers (independent)
git checkout main
git checkout -b french-numbers
echo "print('un')" > french.py
echo "print('deux')" >> french.py
echo "print('trois')" >> french.py
git add .
git commit -m 'Add French number printer'
```

Each branch stems from `main`, and they can be merged in any order without conflicts. Simple, clean, beautiful.

But reality is rarely this neat. More often, you'll have a sequence like: Task #1 fixes a bug, Task #2 refactors the buggy code, and Task #3 implements a new feature using that refactored code. Each depends on the previous one.

## Level 1: Dependent PRs (The Stack Begins)

When PRs naturally depend on each other, you create a stack. Each branch builds on top of the previous one:

```bash
# Branch 1: Base functionality
git checkout -b integers
echo "print('1')" > integers.py
echo "print('2')" >> integers.py
echo "print('3')" >> integers.py
git add .
git commit -m 'Add integer printer'

# Branch 2: Builds on branch 1
git checkout -b rationals
echo "import integers" > rationals.py
echo "print('0.5')" >> rationals.py
echo "print('-100')" >> rationals.py
git add .
git commit -m 'Add rational number printer'

# Branch 3: Builds on branch 2
git checkout -b reals
echo "import rationals" > reals.py
echo "print('pi')" >> reals.py
git add .
git commit -m 'Add real number printer'
```

Your commit history now looks like this:

```
main -> integers -> rationals -> reals
```

Each PR should still focus on a single domain or concern. The `integers` PR might be reviewed and merged first, then `rationals`, then `reals`. When merging, you simply merge from the bottom of the stack upward.

The key principle: keep each PR focused and reviewable on its own, even though it depends on the previous one.

## Level 2: Squash Merges (Where Things Get Interesting)

Here's where most developers run into trouble. With AI-assisted coding, it's normal to accumulate many commits in a single PR. I've had PRs with over 40 commits (don't judge). You want a clean main branch history, so you use squash merges, but this creates a problem for stacked PRs.

When you do `git merge branch-1 --squash`, git creates a new commit with a new hash. Your carefully constructed stack is now broken because `branch-2` and `branch-3` still reference the old commits from `branch-1`.

Here's how to fix it:

```bash
# Merge the bottom PR with squash
git checkout main
git merge --squash branch-1
git commit -m 'Add integer printer'

# Rebase the rest of the stack onto the new main
git checkout branch-3  # Start from the top of the stack
git rebase --update-refs --onto branch-1 main
```

Let's break down that rebase command:

- `--onto branch-1`: The new base for your commits
- `main`: The old base (what you're moving away from)
- `--update-refs`: This is the magic flag that updates all branch references in your stack

Without `--update-refs`, git would rebase your commits but leave your branch pointers at the old commits. With it, all branches in your stack automatically point to the newly rebased commits.

### Pushing Your Updated Stack

Now you need to update your remote branches. You could use:

```bash
git push --force-with-lease --all
```

But be careful: this pushes *all* local branches. If you have other work in progress, you might not want that.

The safer option is to specify branches:

```bash
git push --force-with-lease origin branch-1 branch-2 branch-3
```

But typing all those branch names is tedious. Here's a helper script to automate it:

```bash
#!/bin/bash
# Save as: git-stack-list (in your PATH, e.g., /usr/local/bin)

# Lists all branches in the current stack
# Usage: git stack-list [base_branch]

set -e

BASE_BRANCH="${1:-main}"

STACKED_BRANCHES=$(git for-each-ref --format='%(refname:short)' refs/heads/ \
    | grep -v "^$BASE_BRANCH\$" \
    | while read branch; do
        if git merge-base --is-ancestor "$branch" HEAD 2>/dev/null && \
           git merge-base --is-ancestor "$BASE_BRANCH" "$branch" 2>/dev/null; then
            commit_count=$(git rev-list --count "$BASE_BRANCH..$branch")
            echo "$commit_count $branch"
        fi
    done \
    | sort -n \
    | cut -d' ' -f2)

echo "$STACKED_BRANCHES"
```

Make it executable:

```bash
chmod +x /usr/local/bin/git-stack-list
```

Now you can push your entire stack with:

```bash
git push --force-with-lease origin $(git stack-list)
```

Much better. This script finds all branches that are ancestors of your current branch and descendants of main, sorts them by commit count (bottom to top), and outputs them as a space-separated list.

## Level 3: Handling Changes in the Middle of the Stack

Code review feedback is inevitable. What happens when a reviewer requests changes to a PR in the middle of your stack?

**Best case scenario:** Your PRs modify different files. This is why keeping PRs focused matters. If `branch-1` touches `auth.py` and `branch-2` touches `api.py`, you can make changes without conflicts.

The workflow:

```bash
# Make changes to the middle branch
git checkout branch-1
# ... make your changes ...
git add .
git commit -m 'Address review feedback'

# Rebase everything above it
git checkout branch-3  # Top of the stack
git rebase --update-refs branch-1
```

This replays all commits from `branch-2` and `branch-3` on top of the updated `branch-1`.

**Worst case scenario:** Changes cause conflicts. You'll need to resolve them during the rebase. This is why file separation is so important in stacked PRs.

## Bonus: Automating with GitHub Actions

You can automate stack updates using GitHub Actions. When a PR at the bottom of a stack is merged, automatically rebase and update all dependent PRs.

Here's a basic workflow:

```yaml
name: Update Stacked PRs

on:
  pull_request:
    types: [closed]

jobs:
  update-stack:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Find dependent PRs
        id: find-deps
        run: |
          # Find all open PRs that have this PR's branch as base
          BASE_BRANCH="${{ github.event.pull_request.head.ref }}"
          gh pr list --base "$BASE_BRANCH" --json number,headRefName --jq '.[] | .headRefName' > dependent_branches.txt

      - name: Rebase dependent PRs
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          
          while read branch; do
            git checkout "$branch"
            git rebase "${{ github.event.pull_request.base.ref }}"
            git push --force-with-lease origin "$branch"
          done < dependent_branches.txt
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This is a simplified version. In production, you'd want error handling, conflict detection, and notifications when automatic rebasing fails.

## Tools That Can Help

While this guide focuses on understanding the mechanics, several tools can streamline stacked PR workflows:

- **[Graphite](https://graphite.dev)**: Provides CLI tools and a dashboard for managing stacks
- **[Jujutsu (jj)](https://github.com/martinvonz/jj)**: A Git-compatible VCS with first-class support for stacked changes
- **[git-branchstack](https://github.com/krobelus/git-branchstack)**: Lightweight tooling specifically for branch stacks

These tools automate much of what we've covered, but understanding the underlying mechanics helps when things go wrong (and they will go wrong, because git).

## Best Practices for Stacked PRs

1. **Keep PRs small and focused**: Each PR should be reviewable in 10-15 minutes. If it's larger, it should probably be a stack.

2. **Maintain clear boundaries**: Each PR should modify distinct files when possible. If PRs must touch the same files, keep changes to different functions or sections.

3. **Write good PR descriptions**: Explain the stack structure. "This PR depends on #123" saves reviewers from confusion.

4. **Use draft PRs strategically**: Mark dependent PRs as drafts until their base is merged. This signals they're not ready for full review.

5. **Communicate with your team**: Stacked PRs require more coordination. Make sure reviewers understand the stack structure.

6. **Don't stack too deep**: More than 4-5 PRs in a stack becomes unwieldy. If you're going deeper, consider whether you're breaking changes down appropriately.

## Conclusion

Stacked PRs aren't a silver bullet, but they're a powerful technique for maintaining velocity without sacrificing code quality. The key is understanding the underlying git mechanics so you can handle the inevitable complications.

The workflow becomes second nature after a few stacks. You'll find yourself naturally thinking in terms of dependencies and breaking work into reviewable chunks. And when someone asks "Why don't you just wait for the first PR to merge?", you can smile knowingly and keep shipping.

Just remember: with great stacking power comes great rebase responsibility. Use `--force-with-lease`, never `--force`, and may your merge conflicts be few and far between.

## References & Acknowledgements

- [Stacked Diffs Versus Pull Requests](https://jg.gg/2018/09/29/stacked-diffs-versus-pull-requests/) by Jackson Gabbard
- [Stacked PRs](https://stacked.dev) by Graphite - the original inspiration for this guide
- The countless developers who've struggled through git rebase conflicts so we don't have to

---

*Found this helpful? Bookmark it for the next time you're explaining stacked PRs to a teammate. Found an issue? The author probably needs to rebase something.*
