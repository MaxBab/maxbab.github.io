---
title: "Gerrit Reference Links"
tags: gerrit
---
When I just started to work with `Gerrit` system, the links that I needed to use when pushing changes, confused me.
So I went to the `Gerrit` documentation.  
Would you like to know what I found?  
If yes, follow me to the post...

As part of my daily work with OpenStack, I use `Gerrit` system for the reviews.
From my point of view, the system gives great features for the code review process.

### How Gerrit works
Gerrit has the following structure.

{% include image image_path="gerrit_structure.png" caption="The image taken from the [Gerrit documentation](https://gerrit-review.googlesource.com/Documentation/intro-how-gerrit-works.html)" %}

>When Gerrit is configured as the central source repository, all code changes are sent to **Pending Changes** for others to review and discuss.
When enough reviewers have approved a code change, you can submit the change to the code base.  
* <ins>From the Gerrit documentation:</ins>

### The git-review command  
In my day-to-day work with Gerrit, I'm using the `git review` command line tool.  
The [git-review][git-review] tool has been created and maintained by the OpenStack Foundation. The tool wraps the actual Gerrit command and gives the user an easy and nice experience.  
The actual push command could be viewed by passing the '-n' argument.

git review -n  
<br>
Please use the following command to send your commits to review:  
&nbsp;&nbsp;&nbsp;git push gerrit HEAD:refs/publish/master/new_patch
{: .notice--info}

### Gerrit push
In order to create a new patch, the following push command is actually used:

```bash
# In case there is no remote set.
git push ssh://sshusername@hostname:29418/projectname HEAD:refs/for/branch

# In case remote set to gerrit
git push gerrit HEAD:refs/for/branch
```

The Gerrit push command has multiple options, like set different states of the patch, adding reviewers, etc... All the options could be found in [gerrit push documentation][gerrit_push].

Ok, but why the push link is so different from the git push link?

### Gerrit refs/for namespaces
Gerrit system has an internal logic which uses the refs/for namespaces.  
As mentioned above, the Gerrit push command looks like the following:

```bash
# Push to the repository 'master' branch.
git push gerrit HEAD:refs/for/master

# Push to the review patch reference.
git push gerrit HEAD:refs/for/new_patch
```

**Note**, that the structure of the namespace `refs/for/[BRANCH_NAME]` allows Gerrit to decide if you are pushing directly to the repository (by using an existing branch), or creating a review patch reference.

In addition, Gerrit supports full and short reference to the branch.  
The following command is the same.

```bash
git commit
git push origin HEAD:refs/for/master

git commit
git push origin HEAD:refs/for/refs/heads/master
```

The `refs/for` patch mapping actually mask the internal mechanism of Gerrit from the git client.
The git client thinks that each new patch set pushed to the same commit, but in reality for each new patch set, a new `ref` created under the `refs/changes` namespace.  
This kind of mechanism allows Gerrit to track new commits for the same patch.

The reference for the changes has the following format:
* refs/changes/[CD]/[ABCD]/[EF]
  * [CD] - last two digits of the change number
  * [ABCD] - change number
  * [EF] - patch set number

For example:  
refs/changes/20/884120/1
{: .notice--info}

The patch with the required patch set could be fetched with the following command:

```bash
git fetch https://[GERRIT_SERVER_URL]/[PROJECT] refs/changes/[XX]/[YYYY]/[ZZ] && git checkout FETCH_HEAD
```

### Bypass review
But what happens if I need to bypass the review process?  
Why would I bypass the review?  
It could be done for the following tasks:
* Create a new branch
* Create an annotated tag for the release
* Rewrite branch history by using the force-update

I'm using the bypass review for the `master` branch update and annotated tags creation.  
For more details, refer to my other [post][tag_git_version].

The direct pushes that bypass the review restricted by the Gerrit to:
* refs/heads/* - any branch can be created, deleted or updated
* refs/tags/* - creation of the annotated tags

### Finally
I hope that at this step of the post, the `Gerrit` namespaces looks less confusing for you.  

[git-review]: https://linux.die.net/man/1/git-review
[gerrit_push]: https://gerrit-review.googlesource.com/Documentation/user-upload.html#_git_push
[tag_git_version]: {% post_url 2019/2019-03-12-tag-git-version-in-gerrit %}
