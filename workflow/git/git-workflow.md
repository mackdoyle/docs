#Git Workflow Policies and Best Practices

*Please Read: [Feature Branch Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow) as well as [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) to better understand this practice.*


##Creating a Branch
To start working on a task, first create a feature branch.  The Master branch is restricted so always create your feature branch from Staging.

Fetch a new remote `staging` branch by running:

```bash
git fetch origin
git checkout staging
```
Then create a new feature branch:

```bash
git checkout -b <my_feature_branch> staging
```

Commit your work as normal to you local feature branch.


##Merging into Master
After your work is done, you can merge your feature branch into staging, push the changes and delete your feature branch.

```bash
# Move to the staging branch
git checkout staging
# Marge your feature branch into staging
git merge --no-ff <my_feature_branch>
# Delete your feature branch (optional)
git branch -d <my_feature_branch>
# Push the staging branch to the remote repository
git push origin staging
```

When everything is done, notify us that your feature branch has been merged into staging into staging and we will pull it onto the staging server for testing. Once there, we will test the changes and upon approval, merge into master and pulled onto the production server.
