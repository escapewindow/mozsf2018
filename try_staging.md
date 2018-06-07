# Staging releases on Try

## Overview

We want our staging releases to be testable on Try. There are two phases here:

## Phase 1: All task types run on Try

Tom Prince has most of this working due to his work migrating Thunderbird to Taskcluster. However, a) I'm not sure if the entirety of the `promote`/`push`/`ship`/`promote_rc`/`ship_rc` phases are covered, and b) there are a number of Firefox and Fennec specific tasks types that also may not be covered.

The first part of this phase would be to determine which task types we're missing for Firefox and Fennec Try staging releases and get them runnable on Try.

### Phase 1.5: all task types return useful info on Try

Just because some of the tasks run on Try doesn't mean they return useful information. For instance, if a pushapk task runs on Try as a dummy task that runs `/bin/true` and is perma-green, we don't necessarily have any assurances that changes to the pushapk task, script, or pool will work when we land our changes in production.

Ideally, all release task types will give us *some* useful information when we run them, even on Try.

#### pushapk

Since [bug 1412836](https://bugzilla.mozilla.org/show_bug.cgi?id=1412836), there's a special pushapk instance compatible with the [dep] key. When detected, it performs all the things done by the production instance, but connecting to Google Play. Thus, this instance doesn't have any access/account that might mess up the production data.

:warning: This hasn't been tested on try. There might be an integration bug due to configuration.

#### pushsnap

Like pushapk, since [this commit](https://github.com/mozilla-releng/pushsnapscript/commit/3a8eb4c3de92403b0b5e202a550efe69c79f8d01), there's a special pushsnap instance that doesn't contact the snap store. However, no checks on the binary is done yet. Therefore, this task is mainly a no-op.

:warning: This hasn't been tested on try. There might be an integration bug due to configuration.

#### treescript

- treescript - we could potentially bump the version and tag Try. We'll likely need to update the cloning logic to pull only the specific Try head. We probably want to add a level 1 secret ssh key that only has level 1 commit privileges.

#### bouncer
- bouncer-{check,aliases} - these, and possibly final verify, tend to break on staging releases, I think because of a shared staging bouncer instance and different release history between prod and staging. This may also affect final verify.

#### Update Verify

These are perma-red in staging. We need to allowlist the try- channel names if we change them, and we need to deal with the fact that we're updating to a dep-signed binary rather than a release-signed one. The updater itself has a hardcoded set of allowed signing keys which doesn't include the dep key; we need to find a solution here if we want useful info from these tasks. And these tests depend on the shared staging balrog instance being in a specific state; parallel staging releases can easily break the testing assumptions, which segues nicely into phase 2.

#### potentially others

This list may not be complete; we should verify that all task types give useful information on Try.

## Phase 2: Concurrent staging releases on Try

In my mind, we need to be able to run concurrent staging releases on Try before we're done with this project.

### The problem

Currently, staging releases rely on the state of the bouncer, balrog, shipit staging servers; concurrent staging releases can break that expected state for the other staging releases. We don't have the capability of deleting files from the staging S3 bucket, so the state of the bucket is also in question. Enough of our workflow is based upon the version and channel in-tree that we're seeing staging version inflation. For example, if we need to test RC behavior, we have to have a version that ends in .0; because we've already tested 62.0 and 63.0, we'd have to test with version 64.0.

Restricting ourselves to a single staging release has been an acceptable limit as long as staging releases were restricted to the releng team, though we sometimes stepped on our own toes running concurrent tests on maple and jamun. If we can each push to try but can only run our staging releases synchronously, we lose a lot of the utility and flexibility that comes with Try.

Two ways we could address this is with namespaces or a library of staging services, described below.

### Namespaces

If we allowed for staging release namespaces, and changed the staging services to use those namespaces, we could avoid collisions. For example, if I created an `aki-rc-20180606` namespace, we could use the `aki-rc-20180606-beta` update channel, publish to the staging bucket under `candidates/aki-rc-20180606/` directory, and/or possibly use the `61.0b1.aki-rc-20180606` version number after un-coupling the release behavior from the version number. Ideally we could allow for concurrent staging releases that don't affect the others' behavior, though it could get a little messy.

### Library of staging services

We brainstormed an idea to automatically launch new balrog/bouncer/s3 staging docker instances when we start a new staging release, and tearing it down afterwards. This is a bit complex, and doesn't allow for more complex tests, like a series of staging releases where we test updating from one to the next. It may make more sense for us to be able to request new instances beforehand, like checking out books from a library, and return them when we're done.

#### Balrog instance

We talk about re-setting the staging balrog instance to something closer to the production instance, to make tests closer to production. We could automate a process to spin up a new balrog docker instance, populate it with recent production data, and allow for staging balrog scriptworkers to access it... an in-tree patch could point the graph at that instance.

Since we're checking this instance out, rather than auto-spinning it up, we then have the ability to make any changes to that instance, should we want to test a separate configuration.

We can use this instance as long as it's useful, and then retire it.

Since these instances are separate, we don't have to worry about changing channel names to avoid cross-staging-release interference. We could just use `beta` or `release` or whatever we're testing.

#### S3 docker instance

We have to consider the stage of the S3 staging bucket when we start a new staging release. (I believe we default to downloading previous releases from the release bucket to craft and test updates; we also have to pay make sure we don't reuse version numbers or build numbers to avoid push-to-cdns bustage.)

We were brainstorming spinning up a docker instance that can mock the S3 api. We can beetmove our artifacts to that instance; we can either point at the production bucket or this instance, with redirects, for downloading previous releases for update generation and testing.

Similar to the Balrog instance, we could make any desired changes to the environment before starting our release, point the tree and staging balrog scriptworkers at it via in-tree taskgraph changes, use it for as long as we need it, and then retire it. Since we can each have our own S3 docker instances, we don't have to worry about cross-staging-release interference.

#### Bouncer instance

Similarly, to avoid depending on a shared bouncer state, we can spin up an independent bouncer instance to use for our staging releases.

#### Ship-it instance?

We probably need either to make ship-it able to support multiple staging environments, or we may want to spin up multiple ship-it instances.

#### Others?

This may not be a comprehensive list of services we need to support.

### Some other solution?
