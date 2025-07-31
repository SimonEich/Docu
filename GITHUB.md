# github

## New Project
- echo "# IMG_Resize" >> README.md
- git init
- git add README.md
- git commit -m "first commit"
- git branch -M main
- git remote add origin https://github.com/SimonEich/IMG_Resize.git
- git push -u origin main
- git add .
- git commit -m 'start'
- git push

## Branches
### New Branch
#### Create in VS Code Terminal
- `git branch <Branch name> ` create a new Branch
- `git checkout -b <new_branch_name>` Change to new Branch/If Branch not exists, this will create the Branch
- `git branch` Checks in which is the current Branch
- `git checkout main` Switching to another Branch (main)
### Merge Branch
- `git checkout main` Merge a Branch into Main
- `git merge your-branch-name` If there are no conflicts, Git will merge the changes automatically.
- `git push origin main` After merging, push the updated main branch.
- `git push -u origin your-branch-name` git push -u origin your-branch-name

## Pull/Push
### Clone Respository
##### Official Website
- `https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository?platform=mac`
##### Step by Step
- On GitHub, navigate to the main page of the repository.
- Above the list of files, click  Code.
- Copy the URL for the repository.
- `git clone https://github.com/YOUR-USERNAME/YOUR-REPOSITORY`
