---
layout: post
title: "Another git branching model"
description: "This git branching model is used on my current project and is heavily based on github workflow"
date: 2013-09-08 18:42
comments: true
tags: [git, github, development workflow]
---

Today I'll describe git branching model we use on our project.

#### Prehistory

We had tried [git-flow](http://nvie.com/posts/a-successful-git-branching-model/), but it appeared to be too complicated for our small team.
So after thinking a while we decided to build up our own workflow, that will perfectly match our [continuous delivery](http://en.wikipedia.org/wiki/Continuous_delivery) needs.
As our basis we took [github flow](http://scottchacon.com/2011/08/31/github-flow.html) which is extremely simple and effective.

<!-- more -->

#### Before we start

This workflow focused on rapid deployment of any feature as soon as its ready. So if you have slow and complex QA processes this workflow may not work well for you.

__For those who don’t want to read long text__ there is TL;DR paragraph at the end.

#### Adjustments to github-flow

Github workflow rules look like this:

1. Anything in the ```master``` branch is deployable
2. To work on something new, create a descriptively named branch off of ```master``` (ie: ```new-oauth2-scopes```)
3. Commit to that branch locally and regularly push your work to the same named branch on the server
4. When you need feedback or help, or you think the branch is ready for merging, open a [pull request](https://help.github.com/articles/using-pull-requests)
5. After someone else has reviewed and signed off on the feature, you can merge it into master
6. Once it is merged and pushed to ```master```, you can and should deploy immediately

Lets go through each rule.

1. Perfect rule! ```master``` branch should always be stable enough to be deployed to production.
2. We felt small lack of connection between issue and branch name, so we took slight change on naming conventions, so our branches named corresponding to tracker issues (ie: ```12345```). Lets call those branches as "feature branches"
3. No changes here. every feature branch is a workplace. Code can be unstable or incomplete here.
4. We open pull requests only when feature developer thinks he's done with it. Most of code discussions usually done via IM or verbally,
so at the time developer creates a PR it should be 90% done. After PR got created, CI server runs tests on feature branch
other developers review and comment on the code, QA team verifies the ticket.
Developers are responsible to keep feature branch up-to-date with master to minimize possibility of conflicts.
5. After all conditions of point 4 are met product owner/delivery manager/whoever_responsible_for_delivery can merge the branch into master.
Its highly recommended to do it as soon as possible, so master didn't diverged from feature branch too much.
In 99% product owner will be happy to merge and deploy feature as soon as its ready, so he'll merge the branch without delay.
Development team shouldn't merge feature branches since we want product owner to have an ability to deploy only those features that he wants to be deployed.
If you have slow QA team your feature branches could diverge from master too much while QA team does their testing. This fact can lead to issues after merge to master.
6. Immediate deploy can be something business side of project would not appreciate, so we kept the right to deploy to product owner.
Once it merged development team should think of it as "deployed". Real deploy can happen whenever product owner decides to do that.

#### Handling large feature packs

Besides small independent features there can be large packs of features that doesn't make much sense without each other. We call them "epics".

Lets say our project has redesign planned in 15 tickets. For some reason business side of the product wants to deploy
all these features in one go, so we can't deploy them independently as soon as each of them ready.
First person who starts to work with ticket related to "epic" change starts new epic branch (ie: ```epic/app_redesign```).
This "epic" branch should be treated like temporary master for all features related to app redesign.
Still, you need to base all your branches on current ```master```, but your pull requests should point to "epic" branch.
After all tickets in "epic" are done we can create PR to master branch and do final review\CI\QA cycle and merge it.

#### For those who don't want to read long text above

Here are all steps summarized.

1. ```master``` branch should be always deployable to production.
2. To work on something new, create a branch off of ```master``` using feature ticket number (ie: ```feature/12345```)
3. Commit to that branch locally and regularly push your work to the same named branch on the server
4. Whenever you think you finished the feature - create a PR to ```master``` or to "epic" branch in case of "epic" related feature. (see "Handling large feature packs")
5. After someone else has reviewed the code and QA team signed off on the feature, product owner can merge it into ```master```
6. Product owner can deploy master to production at any given time.


#### Workflow enchancements

* Its helpful to have automated mails via github hooks for PRs with unresolved conflicts. That will help development team to react faster.
* Deployment script that will deploy feature-branch + latest master will be helpful for QA team.
* Github + CI bots help to track CI build status on each feature branch.


