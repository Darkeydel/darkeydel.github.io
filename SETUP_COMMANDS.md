# GitHub Pages Setup Commands

## Commands performed to create the security research blog:

```bash
# Check GitHub CLI auth
gh auth status

# Initialize git and commit files
git init
git add .
git commit -m "Initial commit - Security research blog"

# Create GitHub repository and push
gh repo create darkey-del.github.io --public --source=. --push

# Create GitHub Actions workflow directory
mkdir -p ".github/workflows"

# Add, commit and push workflow
git add .github/
git commit -m "Add GitHub Actions workflow"
git push origin master

# Check workflow status
gh run list --repo darkey-del/darkey-del.github.io
gh run view 22282659541 --repo darkey-del/darkey-del.github.io --log-failed

# Remove submodules causing build failure
git rm --cached agent-skills impossible-leak
rm -rf agent-skills impossible-leak HTTP2-connect_playground "Parsar+Differential+leadingtoAVuln"

# Commit cleanup and rename branch
git commit -m "Clean up - keep only blog files"
git branch -M main
git push origin main --force

# Fix workflow branch
git add .github/workflows/jekyll.yml
git commit -m "Fix workflow branch"
git push origin main

# Push to master branch for GitHub Pages
git push origin main:master
```

## To add new blog posts:

```bash
# Add new post to _posts/ directory
git add .
git commit -m "New blog post"
git push origin main master
```

## Fixing username change issue:

```bash
# Check repos under new username
gh repo list Darkeydel

# Rename repo to match new username
gh repo rename darkeydel.github.io --repo Darkeydel/darkey-del.github.io

# Update git remote URL
git remote set-url origin https://github.com/Darkeydel/darkeydel.github.io.git

# Verify remote
git remote -v

# Push to trigger rebuild
git push origin main
```
