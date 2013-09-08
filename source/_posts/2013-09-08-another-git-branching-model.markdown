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

We tried [git-flow](http://nvie.com/posts/a-successful-git-branching-model/) and it appeared to be too complicated for our small team.
After thinking for a while we built up our own workflow that perfectly matches our [continuous delivery](http://en.wikipedia.org/wiki/Continuous_delivery) needs.
As a basis we took [github flow](http://scottchacon.com/2011/08/31/github-flow.html) which is extremely simple and effective.

<!-- more -->

#### Before we start

This workflow is focused on rapid deployment of any feature as soon as it's ready. Hence, If you have slow and complex QA process this workflow may not work well for you.

__For those who don’t want to read long text__ there is TL;DR paragraph at the end.

#### Adjustments to github-flow

Github workflow rules look like this:

1. Anything in the ```master``` branch is deployable
2. To work on something new, create a descriptively named branch off of ```master``` (ie: ```new-oauth2-scopes```)
3. Commit to that branch locally and regularly push your work to the same named branch on the server
4. When you need feedback or help, or you think the branch is ready for merging, open a [pull request](https://help.github.com/articles/using-pull-requests)(PR)
5. After someone else has reviewed and signed off on the feature, you can merge it into ```master```
6. Once it is merged and pushed to ```master```, you can and should deploy immediately

Lets go through each rule.

1. Perfect rule! ```master``` should always be stable enough to be deployed to production.
2. We felt little lack of connection between issue and branch name, so we made slight change
of naming convention. We name branches according to tracker issue numbers (ie: ```12345```).
Lets call these branches "feature branches"
3. No changes here. Every feature branch is a workplace. Code can be unstable or incomplete here.
4. We open pull request only when feature developer thinks he's done with the feature. Most of
code discussions are done via IM or verbally. After PR have been created CI server runs tests on a feature branch,
other developers review and comment the code, QA team verifies the ticket.
Developers are responsible to keep feature branch up-to-date with ```master``` to minimize
possibility of conflicts.
5. After all conditions of point 4 have been met product
owner/delivery manager/whoever_responsible_for_delivery can merge the branch into ```master```.
It's highly recommended to do this as soon as possible, so ```master``` doesn't diverge from feature branch too much.
In 99% product owner is happy to merge and deploy feature as soon as it's ready.
Development team shouldn't merge feature branches since we want product owner to have
control on what should be deployed.
6. Immediate deploy can be the thing that is not appreciated by business side of the product,
so we keep the right to deploy to product owner. As soon as feature branch is merged development
team should think of it as "deployed". Real deploy can happen whenever product owner decides to do it.

#### Handling large feature packs

Besides small independent features there can be a large pack of features that doesn't make
much sense without each other. We call such packs "epics".

Lets say our project has a redesign planned in 15 tickets. For some reason business side of
the product wants to all these features to be deployed at once, so we can't deploy them as
soon as each of them is ready. First person, who starts to work with ticket related to "epic",
starts new epic branch (ie: ```epic/app_redesign```). This "epic" branch should be treated
like temporary ```master``` for all features related to the app redesign. Your pull requests
should point to "epic" branch. However, you need to base all your branches on ```master```.
After all tickets in "epic" are done we can create PR to ```master```, do final
review\CI\QA cycle and merge it.

#### For those who don't want to read long text above

Here are all steps summarized.

1. ```master``` branch should be always deployable to production.
2. To work on something new, create a branch off of ```master``` using feature ticket number (ie: ```feature/12345```)
3. Commit to that branch locally and regularly push your work to the same named branch on the server
4. Whenever you think you have finished the feature create a PR to ```master``` or to
"epic" branch in case of "epic" related feature. (see "Handling large feature packs")
5. After someone else has reviewed the code and QA team signed off on the feature,
product owner can merge it into ```master```
6. Product owner can deploy ```master``` to production any time.


#### Workflow enchancements

* It's helpful to have automated mails via github hooks for PRs with unresolved conflicts.
That will help development team to react faster.
* The script that will deploy feature-branch + latest ```master``` is helpful for QA team.
* Github + CI bots help to track CI build status on each feature branch.


