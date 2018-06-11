# Staging releases on Try

Moved [here](https://docs.google.com/document/d/1_p6jqQ9YUojerRwgfdMhEyjOFpkvkHONDd6nNnRMNlY/edit) for dicussion.

## Overview

We want to be able to run our staging releases on Try. This would allow the Releng team to test arbitrary release configs without having to set up maple or some other project branch first; and it would open release automation testing to other teams that want to make sure their changes haven't broken release automation.

There are two phases here:

## Phase 1: All task types run on Try

Tom Prince has most of this working due to his work migrating Thunderbird to Taskcluster. However, a) I'm not sure if the entirety of the `promote`/`push`/`ship`/`promote_rc`/`ship_rc`/`promote_partners` phases are covered, and b) there are a number of Firefox- and Fennec- specific tasks types that also may not be covered.

The first part of this phase would be to determine which task types we're missing for Firefox and Fennec Try staging releases and get them runnable on Try.

### Phase 1.5: all task types return useful info on Try

Just because some of the tasks run on Try doesn't mean they return useful information. For instance, if a pushapk task runs on Try as a dummy task that runs `/bin/true` and is perma-green, we don't necessarily have any assurances that changes to the pushapk task, script, or pool will work when we land our changes in production.

Ideally, all release task types will give us *some* useful information when we run them, even on Try. We should strive for as much useful information as is practical.

At the end of this phase, releng and other teams should be able to fully test staging releases on Try, with the caveat that only a single staging release per product can run at a time.

#### pushapk

Since [bug 1412836](https://bugzilla.mozilla.org/show_bug.cgi?id=1412836), there's a special pushapk instance compatible with the [dep] key. When detected, it performs all the things done by the production instance, but connecting to Google Play. Thus, this instance doesn't have any access/account that might mess up the production data.

:warning: This hasn't been tested on try. There might be an integration bug due to configuration.

#### pushsnap

Like pushapk, since [this commit](https://github.com/mozilla-releng/pushsnapscript/commit/3a8eb4c3de92403b0b5e202a550efe69c79f8d01), there's a special pushsnap instance that doesn't contact the snap store. However, no checks on the binary is done yet. Therefore, this task is mainly a no-op.

:warning: This hasn't been tested on try. There might be an integration bug due to configuration.

#### treescript

I believe we only support Treescript running on level 3 repos at the moment.

We could potentially bump the version and tag Try. We'll likely need to update the cloning logic to pull only the specific Try head.

#### bouncer

Bouncer-{check,aliases}, and possibly final verify, tend to break on staging releases. I think this is because we use a shared staging bouncer instance, and staging release history is divergent between prod and staging. This may also affect final verify.

I'm hoping we can find some way to make this more workable in staging before phase 2.

#### Update Verify

These are perma-red in staging. We need to allowlist the staging update channel names if they differ from production, and we need to deal with the fact that we're updating to a dep-signed binary rather than a release-signed one. The updater itself has a hardcoded set of allowed signing keys which doesn't include the dep key; we need to find a solution here if we want useful info from these tasks (our previous brainstormed pointed at using the xpcshell? updater, which allows using dep-signed MARs). These tests depend on the shared staging balrog instance being in a specific state; parallel staging releases can easily break the testing assumptions, which segues nicely into phase 2.

#### Version numbers

As alluded to above, we base our behavior on the version number and build number. We also have a fragile partial update situation: aiui, the partial updates specified need to exist in both staging and production, making it trickier to find which version+buildnumber(s) to specify.

For the former, we could un-hardcode version-based behavior, and have an overrideable pull-down menu for beta, release, or RC behavior. For the latter, we could adjust the checks and workflow to allow for production-only releases for partials. We might decide to live with both of these limitations here, but ideally with some documentation.

#### Documentation

At the end of this phase, documentation needs to be enough to reserve a product for staging releases, trigger a staging release on Try, understand what each of the `promote`/`push`/`ship` phases do, and how to troubleshoot. I imagine this will be a living document that grows over time.

#### potentially others

This list may not be complete; we should verify that all task types give useful information on Try.

## Phase 2: Concurrent staging releases on Try

In my mind, we need to be able to run concurrent staging releases on Try before we're done with this project.

When we finish implementing this, we should be able to support `n` concurrent staging releases on Try.

### The problem

Currently, staging releases rely on the state of the bouncer, balrog, and shipit staging servers, as well as the staging S3 bucket; concurrent staging releases cause bustage.

Restricting ourselves to a single staging release at a time (per product) has been an acceptable limit as long as staging releases were restricted to the releng team. However, if we're limited to running our Try staging releases synchronously, we lose a lot of the utility and flexibility that comes with Try.

We could potentially address this with namespaces or a library of staging services, described below.

### Namespaces

Namespaces in shared services could help avoid collisions. Rather than trying to update and test the beta and beta-localtest channels on the staging balrog server, it could be *user*-beta and *user*-beta-localtest. Beetmover could use *user*-candidates/ and *user*-releases/. Bouncer could either use the namespace in the product name or the version number. Ideally we could allow for concurrent staging releases that don't affect the others' behavior. The downside here is it could get quite messy.

### Library of staging services

We may be able to request new docker instances of bouncer, balrog, and "s3" before running a staging release, like checking out books from a library, and retire them when we're done.

We had brainstormed spinning these instances up automatically, but that may prove tricky, time consuming, and not allow for alternate or complex testing scenarios. For instance, we may want to run two or more staging releases in a row, and verify that the first can update to subsequent releases. The library allows us to set up instances beforehand, modify them if desired, keep them up for one or more releases, and choose when to retire them.

#### Balrog instance

With the shared staging balrog server, we occasionally want to reset the instance, to make tests closer to production. We could automate a process to spin up a new balrog docker instance, populate it with recent production data, and allow for staging balrog scriptworkers to access it... an in-tree patch could point the graph at that instance.

We can use this instance as long as it's useful, and then retire it.

Since these instances are separate, we don't have to worry about changing channel names to avoid cross-staging-release interference. We could just use `beta` or `release` or whatever we're testing.

#### S3 docker instance

We currently have to consider the stage of the S3 staging bucket when we start a new staging release. (We can't reuse pushed version numbers to avoid push-to-cdns bustage.)

We were brainstorming spinning up a docker instance that can mock the S3 api. We can beetmove our artifacts to that instance. When we download previous releases for update generation and testing, we can point at this instance, and redirect to production if the files don't exist here.

Similar to the Balrog instance, we could make any desired changes to the environment before starting our release, point the tree and staging beetmover scriptworkers at it via in-tree taskgraph changes, use it for as long as we need it, and then retire it. Since we can each have our own S3 docker instances, we don't have to worry about cross-staging-release interference.

#### Bouncer instance

Similarly, to avoid depending on a shared bouncer state, we can spin up an independent bouncer instance to use for our staging releases.

#### Ship-it instance?

Overall ship-it allows for concurrent releases, with the caveat that it has its constraints around version- and build-numbers. We may want to either bypass this for staging (allow `n` version-buildnumber combinations?) or allow for spinning up new ship-it instances.

#### Others?

This may not be a comprehensive list of services we need to support... we may have to add more to this list.

### Some other solution?
