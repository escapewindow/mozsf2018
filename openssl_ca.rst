Chain of Trust OpenSSL CA
=========================

(discussion moved to `gdocs <https://docs.google.com/document/d/1YAySxchQYp6SBet9DJpfHOU0knO4yFqEXIWuWoO_h4Y/edit>`__)

.. contents::

Tl;dr
-----

For the taskcluster `chain of
trust <http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html>`__,
we need workers to have a non-scopes-governed secret as a second factor.
We also need a way to manage these secrets.

We currently use a homegrown GPG solution, but our implementation
requires a lot of manual maintenance, is more fragile than we want, and
`requires an EOLed version of
GPG <https://github.com/mozilla-releng/scriptworker/issues/124>`__.

We've been brainstorming an OpenSSL CA solution for more manageable
secrets.

Current GPG solution
--------------------

Aiui, the Taskcluster and Relops teams create keypairs for docker-worker
and windows generic-worker, respectively. They add the public key to the
`cot-gpg-keys repo <https://github.com/mozilla-releng/cot-gpg-keys>`__,
which a member of Releng needs to merge and tag with a gpg signature.
These keys are shared across level 3 AMIs and images, and are only
differentiated by worker implementation. The docker-worker key is
generated at AMI creation time; the generic-worker key is populated from
a secrets repo at image creation time.

The Releng team creates keypairs for each scriptworker instance and
signs each key with a trusted key. The pubkeys go into the
``scriptworker/valid/`` directory; the pubkey for the trusted keys are
in the ``scriptworker/trusted`` directory. We add the signed
keypair to releng puppet hiera, where it gets distributed to the scriptworkers.

The `cot-gpg-keys
repo <https://github.com/mozilla-releng/cot-gpg-keys>`__, the
`scriptworker.gpg module <https://github.com/mozilla-releng/scriptworker/blob/master/scriptworker/gpg.py>`__
contains the code for scriptworker gpg usage, both for maintaining
multiple gpg homedirs and for key and signature verification. We use
`puppet-triggered gpg homedir
rebuilds <https://hg.mozilla.org/build/puppet/file/d90611235731/modules/scriptworker/manifests/chain_of_trust.pp#l61>`__
to update the keys, every time the latest signed git tag changes. GPG
key management is described
`here <http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html#chain-of-trust-gpg-key-management>`__.

Current pain points
~~~~~~~~~~~~~~~~~~~

* Taskcluster / relops teams need to use more-manual-than-desired process to generate gpg keys and copy the public key into a cot-gpg-keys pull request manually.
* Releng team needs to manually generate and sign scriptworker gpg keys.

  * the public and private keys go into Hiera, a manual and error-prone process
  * these keypairs are unique per instance, rather than shared across pools, despite no verification that the key matches the ``workerId``. This means we need to create new gpg keypairs every time we enlarge a scriptworker pool.

* ``scriptworker.gpg`` currently bails on any expiration or revocation markers on any keys or subkeys, to try to catch any issues before they can cause problems. This means a number of users' work GPG keys aren't usable for CoT, because they have subkeys or signatures that have expired or been revoked. We could drop this "feature", but it's unclear if this would have a negative impact on security.

  * ``scriptworker.gpg`` requires GPG 2.0.x behavior. GPG 2.0.x is past its end of life.

* expiration and revocation is managed by the existence of a key in the cot-gpg-keys repo, which requires a manual update. There is no concept of a key that is valid for a given window of time and still verifiable historically.
* because key management is quite fiddly, we only have cot signature verification enabled on the scriptworkers. This means we can't take advantage of the CoT to download and verify artifacts in other tasks.

Initial OpenSSL CA proposal
---------------------------

First, the security team generates a root cert and creates an
intermediate cert to use in operations. The root cert is locked away.

We use the intermediate cert to sign worker certs. (TBD whether this is
per-workerId, workerType, AMI, level, worker implementation, etc.; the process
for requesting signed CSRs is also TBD.)

We'll need worker code to use the new certs to sign the chain of trust
artifact. We currently have worker implementations in Go, Node, and
Python, though the Node implementation may be scheduled for retirement.

We need library code or a tool to verify the signatures of chain of
trust artifacts. For parity, scriptworker will need this ability for
chain of trust verification. However, we could implement libraries or tools that
could verify a chain of trust artifact's signature, downloads task
artifacts, and verifies their shas. (Bonus points for verifying the entire
chain of trust back to the tree.) Given these libraries or tools, e could
extend artifact verification across the entire graph, not just scriptworker
tasks.

I assume we'll trust the root CA. Any worker cert signed by one of
its intermediate CAs will be valid for the lifespan of the worker cert.
We will likely need a certificate revocation list that the verification
libraries or tools will check, to invalidate any certs that leak.

Open Questions
~~~~~~~~~~~~~~

Do we trust the provisioners?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If we trust the provisioners, we could give the AWS provisioner an
intermediate cert for workerTypes under its control. It could sign
worker CSRs, either by workerType or workerId. If we wanted to generate
more, shorter-lived certs, we could potentially embed AWS instance ID
information into the cert as well. Alternately, we could add a set of
pre-generated worker certs to the provisioner, and allow it to
distribute them to the workers it spins up.

Distributing worker certs from the provisioner allows for a more
automated system than our current manual key deployment. My main
question is whether we trust the provisioner to handle an intermediate
cert or a set of worker certs. If the chain of trust is designed to
protect against a scopes leak, does this design weaken that protection?

Do we want worker certs to be shared widely?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At one extreme, we could create a single worker cert, and share that
cert across all level 3 workerTypes. This is easy to maintain,
distribute, and verify; we don't necessarily even need a CA here: a
self-signed cert would likely suffice. However, given an attack on CoT and
a single compromised worker, any workerType would be able to forge the chain of
trust artifact for any other workerType. E.g., if a Windows generic worker were
compromised, the attacker could use its cert to generate fake taskgraph
artifacts that could mark malicious tasks as valid.

At the other extreme, we could generate certs that are only valid for a
given workerId and EC2 instance ID. This means we would need to automate
cert generation and signing, but the compromise of a single cert would
be restricted in the damage it could perform.

How will we sign worker certs?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Related to the above questions.

-  file bug to have security team sign CSR with intermediate CA cert
-  automated via provisioner
-  OCC v2 scriptworker
-  ?

Likelihood of a cot download tool?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This would help solve `bug
1370612 <https://bugzilla.mozilla.org/show_bug.cgi?id=1370612>`__.

1. verify the chain of trust artifact signature
2. verify the chain of trust artifact? trace to tree?
3. download artifact(s), and verify their shas against cot artifact?

If we implement (2), that will be more secure than otherwise. It also
assumes chain of trust verification is standalone and ideally
simplified, which are both worthy goals.

In conjunction with taskcluster- and generic-worker mounts, we could
have end-to-end artifact and cot verification in the graph.

Likelihood of artifact verification as part of the official taskcluster platform?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This would simplify chain of trust verification by a significant amount.
