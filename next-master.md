## Terminology
`stage/test` – environment for testing that is used by QA

`release` – any occurrence of deploying code to production

`series` – release

`mini-release` – on-demand release of one or more code-changes to production. Numbered, follows `major.hotfix` scheme, e.g. 250.0. Where `major` is bumped when we do `mini-release` and `hotfix` is bumped when we do `hotfix` release

`PR` – Pull Request

`feature-branch` - branched off from `next`, contains implementation of Big Feature (see definition below)

## Branches
There are two types of branches. Permanent and temporary.

Permanent branches are `master` and `next`.
Temporary branches are all PR branches that we normally use.

#### `master`
Is a production branch that only Release Manager can commit to. Master uses tags that specify release numbers like `v250.0` where last part reserved for hot-fixes.

E.g. `v250.1` is a hotfix release that hotfixes `v250`

#### `next`
Is a next release branch. Release could be any set of changes that would go live together. `next` is never used to do hotfixes.

## Developer's Workflow

Can be split into 4 categories: hotfix, tiny change, small feature and big feature.

(I used word "feature", but it also can be a "set of changes", please don't feel restricted by the name)

> In general, prefix every public branch you created with `<issue-number>-`, all upper-case, this way it's easier to navigate branches and PRs.

#### Hotfix
Should be done for P0 or sometimes P1 issues only.
* create a hotfix branch from last tagged release on `master`: `git checkout v250 && git checkout -b hotfix-for-v250`, most of the times "last tagged" and tip of the `master` will be same thing, so you would be able to just create it from `master`
  * make sure your branch starts from the following prefix `<issue-number>-HOTFIX-` all upper-case, feel free to add more details after
* open PR to `master`, commit your fixes
  * make sure PR name contains word HOTFIX and add "hotfix" label
* when done ask others to review it
* then ask QA to deploy to Stage and test
* when it's done QA merges in to `master` AND to `next` and tags it `v250.1` and it gets deployed ASAP

#### Tiny Change
* do it directly on `next` (PR is optional)

A good definition of a tiny change is:
  * it doesn't require QA Testing
  * it doesn't require code-review

#### Small Feature / Small Changes

Most of your work will be this way.

* branch of `next`
* open PR to `next` like you normally would and collaborate there
* when PR is ready, code-reviewed and passed your testing merge/rebase it to `next`
  * this could be a rebase, to keep history linear since chances of roll-back are minimal
* ask QA to test it
* do bug-fixing directly on `next`

A good definition of a small feature or small set of changes is:
* it takes 1-3 days to develop
* it touches one isolated part of the app and doesn't require regression testing
* and most importantly, your intuition tells you that there won't be a lot of back-and-forth with QA and chances of rolling this back are minimal (Rollback could happen if say release is going tonight and this feature should not go.)

#### Big Feature

  * branch off from `next`
  * open PR like you normally would and collaborate there
  * keep your branch up-to-date with `next`
  * when PR is ready, code-reviewed, passed your testing and is up to date with `next` ask QA to test it
  (:point_up: this is where this wofklow is different from Small Feature)
  * QA double-ckecks that it's up to date with `next` tests it on `test` environment
  * go trough back-and-forth and bug fixes
  * when it's ready let QA merge it to `next`

So idea is that most of the feature testing would happen on `test` environment. And two features A and B could be tested in parallel, we would just need to have two `test` environments. Then both, feature A and B would get merged to `next` and only automated and smoke testing performed before the release.

## QA's Workflow

Only you have direct access to `master`.

#### Pull Requests for mini-releases
As a rule of thumb, for every mini-release open PR from `next` to `master` and call it "Mini Release v250". You will be able to open PR only when `next` has some changes that are not on `master` yet, so a good time for opening would be as soon as anybody commits something after previous release.

Keep this PR open until mini-release is ready to go to production. This way you would have one place to see it all, all the changes that are about to go, all DB scripts, all comments and it will be preserved in history.

When it's ready and is up to date with `master` (see FAQ on how to do that), just fast-forward it to `master` by selecting "rebase and merge" on Github. This way you would help keep code history linear and won't need to wait for another build. Since code in your PR will be identical to code on `master` you can just use assets from last build to deploy to prod.

Sometimes `next` won't be up to date with `master`. The only possible case for this is when `hotfix` got deployed since last `mini-release` and developer forgot to merge it to `next`. In this case you would need to merge `master` into your  `PR`. Use your best judgement if you still want to re-test this or ask engineer who did that hot-fix on what his intuition says.

#### Release Process
To recap the Release Process:

* open PR from `next` to `master` after last release, when few changes were commited to `next`
* when PR is ready fast-forward `master` to it (that would close PR automatically)
* write release notes in Github Releases (https://github.com/Ingenio/keen/releases/new)
* enter corresponding tag there and describe what went into this release briefly in the same way you do it today in Slack
* celebrate :tada:

For hot-fixes it's also a good idea to create a release. Basically do this anytime code goes to production.

#### Testing

You would always test code from `next` on `stage` and code from feature branches on `test`. Doing testing on `test` is a tradeoff that we accept to simplify our workflow. It introduces a small risk that when two or more features that are developed in parallel won't be tested together on `next` in full. Our practical approach in order to keep things simple and to minimize the risks is to do full testing on `test` while keeping feature branch up-to-date with `next`. When testing is done feature branch PR will get merged to `next` as well as other feature branch PRs and before the release you would go through final smoke testing, to make sure it all works after integrating with each other. Automation tests would play a key role in helping with this.

So to repeat, do all testing per engineering request on `stage`, in case of Big Features test corresponding PR on `test` plus do a smoke testing (ideally automated) before sending release to production.

Since it's common to have 2 or more Big Features going in parallel you may need two or more `test` environments. Which should be identical to `stage`.

#### Hotfixes

##### Independently Deployed Hotfix

* deploy hotfix branch to `stage` (ideally we need two `stage` environments to speed things up and to not block ongoing testing on `next`)
* when done merge hotfix PR to `master` + remove hotfix branch (from Github's UI)
* tag `master` with a hotfix release (e.g. last release was 250.0, after hotfix it would be 250.1)
* merge `master` to `next` so everybody will get hotfix as well

##### Hotfix deployed with next `mini-release`
* deploy hotfix branch to `stage`, same as above
* when done testing
  * tag tip of the hotfix branch as `hotfix-250.1` (important step!)
  * merge hotfix PR to `next` + remove hotfix branch (frorm Github's UI)
* follow your regular mini-release process

##### When mini-relase is getting delayed, but hot-fix has to go
It's not uncommon to be overoptimistic about mini-releases and sometimes situation can happen when we thought that hotfix will go with today's mini-release, but it is getting delayed. At the same time hotfix still has to be deployed. In this case you have followed steps above and it's merged it with `next`, so you would need to do:

* checkout that hotfix tag you created above `hotfix-250.1`
* merge it to `master`
* tag `master` with a hotfix release (same as regular hotfix release `250.hotfix-number`)


#### Feature Branches Rollbacks
If two features were developed in parallel, tested on `test` and then merged to `next` it's possible that some crucial things will get discovered that would delay one of the features. But since both were already merged to `next` how would we release feature A without feature B?

Ideally this situation should be pretty rare with this model, since that is why we test feature branches on `test`, if it still happens all you need to do is to ask engineer to roll-back merge commit on `next`. Then re-do your before-release-smoke testing and it's ready to go.

#### Testing 2+ features simultaneously
This is already covered above. You'd deploy `next` to `stage` and use one environment to test multiple things simultaneously. In case of Feature Branches you would use two environments to test 2 features at the same time.


## FAQ

### What about cherry-picking?
It's discouraged and should  be avoided in this model. Cherry-picking is still a useful technique, use it with your own local branches when you want only few commits from your branch on another, but be careful.

### When should I merge PR and when should I rebase?
Always merge Big Features. It's up to you when to merge or rebase a Small Feature PR, I'd go with rebase to keep history simple, since there is almost no value in highlighting when you were done with Small Feature and when it's started, unlike Big Features when it's important.

Always merge hotfixes, since they have historical value.

As always, never rebase public branches (rebasing a PR from Github's UI is the only exception since branch will get removed right away anyway).

### How to check that branch `x` is up to date with `master`?
You can do it from github's UI or from terminal. On Github's UI just pick `x` branch from branches dropdown and you should see something like this, if it's up-to-date:

![Braches up-to-date image](/docs/img/git-branches-up-to-date.png)

If branch is behind same section would say something like:

>This branch is 5 commits ahead, 1448 commits behind master.

From command line:

`git rev-list --left-right --count origin/master...HEAD | sed 's/[0-9]/& behind/' | sed 's/.*/& ahead/'`

Use aliases to simplify your life:

`git config --global alias.up "\!git rev-list --left-right --count origin/master...HEAD | sed 's/[0-9]/& behind/' | sed 's/.*/& ahead/'"`

And then to check if branch `x` is up to date:

`git checkout x && git up`

or to check from any branch with one line, do not use HEAD in above command, bur rather branch name.

### How do I create a tag?

You don't have to do it manually, when you will create a Release on Github it would let you do it automatically.

For all other cases, follow below process.

Make sure you have checked out desired commit, usually it's last commit on master, to make sure:
`git checkout master && git pull`
then
`git tag v250 && git push --tags`
Last command pushes tags to origin.


`npm new mini-release` that would:
* read most recent tag
* increment by 1
* add new one
* push it

and `npm new hotfix` correspondingly
