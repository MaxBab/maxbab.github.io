---
title: "Tag Git Version in Gerrit"
tags: git gerrit
---
When a new set of changes merged into the working branch, you would like to tag it with a specific version.
This will help users to understand what are the changes that have been merged and give the ability to switch to the earlier version of the release in case of issues.

[Semantic versioning][semver] is a common template that guides us on how to tag a new version.
The following article will show the steps that should be accomplished in order to tag a new version in `gerrit`.

In our projects, we are using a [GerritHub][gerrithub] code review system.  
The system integrates with the GitHub repository and accepts patches for the review.
It is important to remember that the synchronization between the GerritHub and GitHub repository done in one way.
All the patches, tags and merges are done in front of the GerritHub system. And the changes are synchronized to the GitHub repository.

> **Git tag** - git tag is an anchor point to a specific commit that does not change. Usually used for the release versions.

Our git repository has two major branches.  
<ins>**Devel**</ins> - All the development work starts from the `devel` branch.
The features branches are created from and merged to the `devel` branch once tested and reviewed.
All the new patches merged into `devel` branch and running for a while in order to make sure there is no errors.  
<ins>**Master**</ins>​ - When required, the changes from the `devel` branch merged into a `master` branch and tagged with a version.

> We are using the **Annotated Tags**.  
For more details, refer to the [git tagging][git tagging] doc.

Please, verify push permissions to GerritHub and (if required) to GitHub.

## So, let's get started!

**Review remotes:**
```bash
$ git remote -v
gerrit ssh://MaxBab@review.gerrithub.io:29418/redhat-openstack/ansible-nfv.git (fetch)
gerrit ssh://MaxBab@review.gerrithub.io:29418/redhat-openstack/ansible-nfv.git (push)
origin git@github.com:redhat-openstack/ansible-nfv.git (fetch)
origin git@github.com:redhat-openstack/ansible-nfv.git (push)
```

**Fetch all changes:**
```bash
$ git fetch --all
Fetching gerrit
Fetching origin
```

**Switch to the ‘devel’ branch and pull the latest changes using rebase:**
```bash
$ git checkout devel
Already on 'devel'
Your branch is up-to-date with 'origin/devel'.

$ git pull --rebase gerrit devel
From ssh://review.gerrithub.io:29418/redhat-openstack/ansible-nfv
* branch    devel -> FETCH_HEAD
Current branch devel is up to date.
```

**Create a new tag:**
```bash
$ git tag -a v1.9.3
```
> The git tag command could be specified with the '-m' argument in order to provide a comment to the tag.  
I'm not using the '-m' option. When the argument is not used, an editor is opened after the tag creation.  
Within the editor, I'm specifying the changes that included within the new tag.

> The created tag could be verified by executing the following command:
```bash
$ git show v1.9.3
```
The result will show the details of the created tag:
- Tag name
- Tagger name
- Date of the tag creation
- Comments of the tag

**Push the new tag to the ‘devel’ branch of the GerritHub:**
```bash
$ git push --follow-tags --no-thin gerrit devel:refs/heads/devel
Counting objects: 1, done.
Writing objects: 100% (1/1), 340 bytes | 0 bytes/s, done.
Total 1 (delta 0), reused 0 (delta 0)
remote: Processing changes: refs: 1, done
To ssh://MaxBab@review.gerrithub.io:29418/redhat-openstack/ansible-nfv.git
* [new tag]  v1.9.3 -> v1.9.3
```
> Note the remote name (**gerrit**) and the name of the branch (**devel**) I'm pushing to.  
The **--no-thin** option is used because of the following possible error: from [stackoverflow][stackoverflow]

**Verify:**
```bash
$ git fetch --all
Fetching gerrit
Fetching origin
```

**Switch to the local ‘master’ branch and pull the latest changes from gerrit 'master' branch using rebase:**
```bash
$ git checkout master
Switched to branch 'master'
Your branch is up-to-date with 'origin/master'.

$ git pull --rebase gerrit master
From ssh://review.gerrithub.io:29418/redhat-openstack/ansible-nfv
* branch   master   -> FETCH_HEAD
Current branch master is up to date.
```

**Update ‘master’ in point to the latest ‘devel’ using merge:**
```bash
$ git merge -s recursive -X theirs devel
Updating e895a8a..a5ce495 (output omitted)
```
> The use of ‘thiers’ argument lets ‘devel’ branch to override possible conflicts on ‘master’

**Verify that the ‘devel’ and ‘master’ branches are aligned with the same code:**
```bash
$ git diff master devel
```

> **Note!** - The output of the previous command should be empty, otherwise fix any diff according to ‘devel’ and amend the merge commit.
```bash
Modify the diff files.
$ git add .
$ git commit --amend -a
```

**In case no 'backports' should be done on the ‘master’ branch, use the following command:**
```bash
$ git push --follow-tags --no-thin gerrit master:refs/heads/master
Total 0 (delta 0), reused 0 (delta 0)
remote: Processing changes: refs: 1, done
To ssh://MaxBab@review.gerrithub.io:29418/redhat-openstack/ansible-nfv.git
e895a8a..a5ce495 master -> master
```
> Note the remote name (**gerrit**) and the name of the branch (**master**) I'm pushing to.  
The **--no-thin** option is used because of the following possible error: from [stackoverflow][stackoverflow]

**In case 'backports' should be done on the ‘master’ branch, --force push is required.**
> The '--force' push argument requires manually pushing to GerritHub (gerrit) and GitHub (origin) in order to keep the sync working.

**Be aware** of the '--force' argument as it will override your repository history.  
**Use it with caution and on your own risk.**
{: .notice--warning}

```bash
$ git push --force --follow-tags --no-thin gerrit master:refs/heads/master
$ git push --force --follow-tags --no-thin origin master:master
```

## Conclusion
At the current state, our repository has an updated branches and created tag, replicated from GerritHub to GitHub repository.  
Enjoy and see you in the next articles!


[//]: # Reference links
[semver]: https://semver.org/
[gerrithub]: http://gerrithub.io/
[git tagging]: https://git-scm.com/book/en/v2/Git-Basics-Tagging
[stackoverflow]: https://stackoverflow.com/questions/16586642/git-unpack-error-on-push-to-gerrit
