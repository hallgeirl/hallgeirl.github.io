---
title:  "Merging git repos into subfolders with history"
date: 2021-03-12 15:00
categories: git
redirect_from:
  - /2021/03/12/merging-git-repos.html
---

Did you know you can merge a git repo into a subfolder of another, while keeping the commit history of the repo you're merging in? It was surprisingly simple. Here's how you do it:
```bash
# Navigate to the repo folder where you're merging into
cd path-to-destination-repo

# Add a remote that points to the repo you're merging in (replace <repo-url> with the actual URL obviously)
git remote add tobemerged <repo-url>

# Fetch changes from the new remote
git fetch tobemerged

# Merge the changes from remote master, but don't commit, and keep the changes from destination.
# This shouldn't actually do anything to the destination repo filesystem.
git merge -sours --no-commit --allow-unrelated-histories tobemerged/master

# Read the files from the repo you're merging in, and put them in the specified subfolder
git read-tree --prefix=subfolder/for/merge/ -u tobemerged/master

# Commit the merge
git commit -m"Merged repos"
```
That's it! You should get the full git log from both repos with `git log`.

I'm not going to take credit for figuring out this all myself - it's based on Eric Lee's answer in this StackOverflow post: 
[https://stackoverflow.com/questions/13040958/merge-two-git-repositories-without-breaking-file-history](https://stackoverflow.com/questions/13040958/merge-two-git-repositories-without-breaking-file-history)