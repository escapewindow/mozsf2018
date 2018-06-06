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

Currently, staging releases rely on the state of the bouncer, balrog, shipit staging servers. We don't have the capability of deleting files from the staging S3 bucket. Enough of our workflow is based upon the version and channel in-tree that we're seeing staging version inflation. For example, if we need to test RC behavior, we have to have a version that ends in .0; because we've already tested 62.0 and 63.0, we'd have to test with version 64.0.

### Namespaces

### Library of resources

#### Balrog instance

#### S3 docker instance

#### Bouncer instance

####
