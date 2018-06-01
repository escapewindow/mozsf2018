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

Pushapk tasks can't submit to the Google Play store, but they might be able to inspect the Try apk and check that it's signed with the expected [dep] key, and has the expected package name / shared ID.

#### pushsnap
pushsnap tasks are similar - we might inspect the binary.

#### treescript
- treescript - we could potentially bump the version and tag Try. We'll likely need to update the cloning logic to pull only the specific Try head. We probably want to add a level 1 secret ssh key that only has level 1 commit privileges.

#### bouncer
- bouncer-{check,aliases} - these, and possibly final verify, tend to break on staging releases, I think because of a shared staging bouncer instance and different release history between prod and staging. This may also affect final verify.

#### Update Verify

These are perma-red in staging. We need to allowlist the try- channel names if we change them, and we need to deal with the fact that we're updating to a dep-signed binary rather than a release-signed one. The updater itself has a hardcoded set of allowed signing keys which doesn't include the dep key; we need to find a solution here if we want useful info from these tasks. And these tests depend on the shared staging balrog instance being in a specific state; parallel staging releases can easily break the testing assumptions, which segues nicely into phase 2.

#### potentially others

This list may not be complete; we should verify that all task types give useful information on Try.

## Phase 2: Concurrent staging releases on Try

### The problem

### Namespaces

### Library of resources

#### Balrog instance

#### S3 docker instance

#### Bouncer instance

####
