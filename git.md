# Git - Global Information Tracker

## 1. Downloading and Installing Git

### Download Git from the following URL and install it with default settings

```
https://git-scm.com/downloads
```

## 2. Checking the Installed Version of Git

```
git --version
```

## 3. Configuring Git

### 3.1 Configure username

```
git config --global user.name "name"
```

### 3.2 Configure email

```
git config --global user.email "email"
```

### 3.3 Configure username and email directly in the config file

```
git config --global --edit
```

1. Press "i" to edit the config file.
2. After making the changes Press "ESC" to save the changes.
3. Type ":wq" to exit the editor.

### 3.4 Check configured username

```
git config --global user.name
```

### 3.5 Check configured email

```
git config --global user.email
```

## 4. Creating a Git Repository

### 4.1 Create a git repository in existing directory

```
git init
```

### 4.2 Create a new directory and git repository

```
git init name
```

## 5. Working with Files

### 5.1 Check the status of the branch

```
git status
```

### 5.2 Add a file to the staging area

```
git add fileName
```

### 5.3 Add all files to the staging area

```
git add .
```

### 5.4 Unstage staged changes

```
git reset
```

### 5.5 Unstage specific files

```
git restore --staged filename
```

### 5.6 Discard unstaged changes

```
git restore .
```

### 5.7 Remove untracked files and directories

```
git clean -fd
```

### 5.8 Untrack all files

```
git rm -r --cached .
```

## 6. Committing Changes

### 6.1 Commit changes

```
git commit -m "message"
```

### 6.2 Check commit log

```
git log
```

### 6.3 Check last commit

```
git show --summary
```

### 6.4 Change last commit message

```
git commit --amend -m "New commit message"
```

### 6.5 Delete last commit (hard reset)

```
git reset --hard HEAD^
```

### 6.6 Delete last commit but keep changes (soft reset)

```
git reset --soft HEAD~1
```

### 6.7 Reset all changes in tracked files after last commit

```
git reset HEAD --hard
```

### 6.8 Hard Reset (most common – discards all commits after that)

This will make `v1` exactly like that commit.

```
git checkout v1
git reset --hard 9dd6c7d2d171e5d1b4367090e792fdefe93bfd41
```

### 6.9 Reset local to a specific commit (destroys everything after it)

```
git reset --hard <commit-hash>
```

## 7. Working with Branches

### 7.1 Check branches

```
git branch
```

### 7.2 Create a new branch

```
git branch branchName
```

### 7.3 Create a new branch and switch to it directly

```
git checkout -b branchName
```

### 7.4 Switch to a different branch

```
git checkout branchName
```

### 7.5 Switch to a different version

```
git checkout hashCode
```

### 7.6 Merge a branch

```
git merge branchName
```

### 7.7 Delete a branch

7.7.1 - Deletes a local branch only if it has been fully merged

```
git branch -d branch-name
```

7.7.2 - Force-deletes a local branch even if it has unmerged changes

```
git branch -D branch-name
```

7.7.3 - Removes the branch from the remote origin

```
git push origin --delete branch-name
```

## 8. Working with Remote Repositories

### 8.1 Clone a repository

```
git clone https://github.com/RitikChahar/Git-GitHub.git
```

### 8.2 Add origin to the git repository

```
git remote add origin https://github.com/RitikChahar/Git-GitHub.git
```

### 8.3 Check origin

```
git remote -v
```

### 8.4 Pull changes from a repository

```
git pull https://github.com/RitikChahar/Git-GitHub.git
```

### 8.5 Pull changes with rebase (avoids merge commits)

```
git pull --rebase
```

### 8.6 Push code to GitHub

```
git branch -M master
git push
```

### 8.7 Push code to a specific branch (same name)

```
git push origin branchName
```

### 8.8 Push code to a specific branch (different name)

```
git push origin localBranchName:remoteBranchName
```

### 8.9 Force push to overwrite remote repository

```
git push -f origin branchName
```

### 8.10 Force push to remote repository

```
git push -f
```

### 8.11 Overwrite remote branch

```
git push origin main --force
```

### 8.12 Mirror code to another repository

```
git push --mirror https://github.com/your-username/new-repo.git
```

### 8.13 Reset local main to match origin/main

```
git fetch origin
git checkout main
git reset --hard origin/main
```

## 9. Working with .gitignore

### 9.1 Create .gitignore file

```
touch .gitignore
```

### 9.2 Add subdirectories to .gitignore

```
**/directory/subdirectory
```

## 10. Git Stash

### 10.1 Stash current changes

```
git stash
```

### 10.2 Stash with a message

```
git stash save "message"
```

### 10.3 List all stashes

```
git stash list
```

### 10.4 Apply the most recent stash (keeps it in stash list)

```
git stash apply
```

### 10.5 Apply and remove the most recent stash

```
git stash pop
```

### 10.6 Apply a specific stash

```
git stash apply stash@{index}
```

### 10.7 Drop a specific stash

```
git stash drop stash@{index}
```

### 10.8 Clear all stashes

```
git stash clear
```

## 11. Advanced Operations

### 11.1 View reference logs (history of HEAD movements)

```
git reflog
```

### 11.2 Delete a git repository

```
rm -fr .git
```
