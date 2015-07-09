 ---
layout: post
title: "Combining Git Repos: The cleanest way without losing history"
description: "Combining Git Repos: The cleanest way without losing history"
category: tutorial
tags: [how-to, git]
---
{% include JB/setup %}

This is a step-by-step guide for consolidating two or more Git repositories. 
The good news is that since a repository in Git is just a directed
acyclic graph, it’s trivial to glue two graphs together and make one
big graph.  The bad news is that there are a few different ways to do
it and some of them end up with a less desirable result (like loosing
commit history etc) than others.

The top stack overflow link on google provides solutions using git
subtree and git submodules which are complex and oriented towards
project structures where one needs to bring in third party code from
another remote and be able to occasionally push-pull etc. Another
side-effect of using git subtree merges is that the all the files from
the new repo are added in a single commit and you lose the valuable
file-level history and blame.

The steps outlined here merges a remote repository (repo-second) to an
existing repository (repo-first). This approach can also be used to
merge two more remote repositories to a newly created repository, if
needed.

Don't worry about screwing up things when following the steps as all
operations would be local until we push, in other words, think twice
before pushing any changes to remote

*Step-by-step guide ( Example: Merging Std to Compute)*

Goto to workspace root on the first repo where the merge will
happen i.e first-repo, and work on a branch

{% highlight bash %}
$ pushd ~/first-repo/
$ git co -b first_repo_branch 
{% endhighlight %}

Add second repository i.e second-repo as remote

{% highlight bash %}
$ git remote add second-repo ssh://git@github.com/second-repo.git
Fetch all commits from second repo
$ git fetch second-repo
...
{% endhighlight %}

Create a local branch from second repo's 'master'

{% highlight bash %}
$ git branch second_repo_branch second-repo/master
Branch second_repo_branch set up to track remote branch master from second-repo.
{% endhighlight %}

Checkout the branch we created in previous step (this will change
your workspace but don't panic.. yet)

{% highlight bash %}
$ git co second_repo_branch
Switched to branch 'second_repo_branch'
Your branch is up-to-date with 'second-repo/master'.
{% endhighlight %}

Move all the files into a desired subdirectory and commit the
changes

{% highlight bash %}
$ mkdir -p src/second-repo
$ git ls-tree -z --name-only HEAD | xargs -0 -I {} git mv {} src/second-repo
 
# git status check before commit
$ git status
 
$ git ci -m"moved second-repo files to src/second-repo subdirectory""

{% endhighlight %}

Merge second repo branch into first repo branch

{% highlight bash %}
$ git checkout first_repo_branch
$ git merge second_repo_branch
{% endhighlight %}

Check project structure and logs with your desired tools. The
consolidated repo now has a merged git graph with the commits and logs
form both second-repo and Compute keeping the same sha1 ids.

{% highlight bash %}
# check logs for a file from second repo
$ git log src/second_repo/second_repo_file
....

# to see all new changes
$ git diff origin/master

# to check outgoing changes(more than you think); this requires a custom script #TODO link)
$ git io
{% endhighlight %}

Repeat steps 2-7 if you have another remote repository to be merged

Celebrate then fix the build (smile)
