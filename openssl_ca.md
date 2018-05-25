Proposal
========

For the taskcluster [chain of trust](http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html), we need workers to have a non-scopes-governed secret as a second factor.

We've been brainstorming an OpenSSL CA solution for these secrets.


Current GPG solution
====================

Currently, we have an ad hoc gpg key management system using the [cot-gpg-keys repo](https://github.com/mozilla-releng/cot-gpg-keys), the [`scriptworker.gpg` module](https://github.com/mozilla-releng/scriptworker/blob/master/scriptworker/gpg.py), and [puppet-triggered gpg homedir rebuilds](https://hg.mozilla.org/build/puppet/file/d90611235731/modules/scriptworker/manifests/chain_of_trust.pp#l61). GPG key management is described [here](http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html#chain-of-trust-gpg-key-management).

Current pain points
-------------------

- Taskcluster / relops teams need to use more-manual-than-desired process to generate gpg keys and copy the public key into a cot-gpg-keys pull request manually.
- Releng team needs to manually generate and sign scriptworker gpg keys.
  - the public and private keys go into Hiera, a manual process
- `scriptworker.gpg` currently bails on any expiration or revocation markers on any keys or subkeys, to try to catch any issues before they can cause problems. This means a number of users' work GPG keys aren't usable for CoT, because they have subkeys or signatures that have expired or been revoked. We could drop this "feature", but it's unclear if this would have a negative impact on security.
- 

Initial OpenSSL CA proposal
===========================

