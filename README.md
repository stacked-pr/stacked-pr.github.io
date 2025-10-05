# Stacked PR

## outline
- overview
  - code review have always been a bottle neck
  - in these age of LLM, its normal for devs to create lots of PRs everyday
  - waiting for PRs to be accepted is counter-productive. however if oftentimes, our next task is dependant on out previously unapproved PR. how to solve it?

## level 0: independent PRs
  - ideal case. each PR inddependent of each others. example is a backend & frontend. BE focus on API implementation, FE focus on providing UI based on agreed upon specification

  ```bash
  # branch 1
  echo "print('one')" > english.py
  echo "print('two')" >> english.py
  echo "print('three')" >> english.py
  git commit -i . -m 'first commit'
  
  # branch 2
  echo "print('unos')" > spanish.py
  echo "print('dos')" >> spanish.py
  echo "print('tres')" >> spanish.py
  git commit -i . -m 'second commit'
  
  # branch 3
  echo "print('one')" > france.py
  echo "print('two')" >> france.py
  echo "print('three')" >> france.py
  git commit -i . -m 'third commit'
  ```
  - however this is not always the situations. example: task #1 focus on bugfix, which required by task #2 which is a refactor, and then task #3 which is the actual feature implementation
  
## level 1: several dependant PRs
  ```bash
  # branch 1
  echo "print('1')" > integers.py
  echo "print('2')" >> integers.py
  echo "print('3')" >> integers.py
  git commit -i . -m 'first commit'
  
  # branch 2
  echo "import numbers" > rationals.py
  echo "print('0.5')" >> rationals.py
  echo "print('-100')" >> rationals.py
  git commit -i . -m 'second commit'
  
  # branch 3
  echo "import rationals" > reals.py
  echo "print('pi')" >> reals.py
  git commit -i . -m 'third commit'
  ```
  - ideally also cleanly separated, and focus on only 1 domain
  - to merge, just merge normally from the bottom most of the stack
  
## level 2: using merge squash
  - in this age of AI-assisted coding, its very likely that we will have many commits on a single PR. in 1 case, I have >40 commits in a single PR. obviously we dont want all of them to be merge into our main branch. we only care about the last result, but we could still mantain the history inside the PR.
  - some reason is for calculating cycle time, which may need detailed commit history
  - however this introduce complication. when we do `git merge <branch_name> -- squash`, it would actually create new commit. so we will need to adjust our history chain with this new commit
  - to do that, from the topmost branch in the stack we could do `git rebase --onto <old_base_branch> main`. if you check it, the commit history have been changed to use the latest squashed commit
  - however if you notice, all the commit hash has also been modified. the implication is that all branch references have now lost
  - to fix it, we should use the `--update-refs` option. so the command is now `git rebase --update-refs --onto <old_base_branch> main`. now all the branches have been changed to point the new hashes
  
  ```bash
  git checkout main
  git merge --squash branch-1

  git checkout branch-3
  git rebase --update-refs --onto branch-1 main 
  ```
  - to update all the remote branches, you could use `git push --force-with-lease --all`. however you should be aware as this would push all branches in your local device. the alternatives is to manually specify all the branches you wish to push:
  
  ```bash
  git push --force-with-lease origin branch1 branch2 branch3
  ```
  - to avoid needing to entering all those branches manually, you could create a small helper script:
  
  ```bash
  #!/bin/bash
  # git-stack-list.sh
    
  # Git custom command to retrieve all branches in a stack
  # Usage: git stack-list [base_branch]
  
  set -e
  
  # Get base branch from argument or default to main
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
  put it in any of your `$PATH` folders. for example you could put it in `/usr/local/bin`. dont forget to run `chmod +x git-stack-list.sh. after that you could push the branches using this command:
  
  ```bash
  git push --force-with-lease origin $(git stack-list)
  ```
## level 3: changes on the PRs in the bottom of the stack
  - ideally again all PRs would edit separate files to avoid conflicts
  - if this condition has been fulfilled, you could just add another commit in the end of the PR you wish to fix. then from the topmost branch again you do `git rebase --update-refs <edited_branch>`. 
## bonus: automation with github action
  >> CREATE SCRIPT FOR UPDATING PR HERE


## tools
- graphite.dev
- jujitsu

## reference & acknowledgements
- stacked.dev by graphite.dev . my original inspiration, but felt that its too shallow. thats why I created this website.
- chatgpt convo: https://chatgpt.com/share/68be45f8-2910-800b-90c6-c17c5e0464a4
- claude convo: https://claude.ai/share/f48b0df2-689c-4b69-aa17-eb050a4645f5
