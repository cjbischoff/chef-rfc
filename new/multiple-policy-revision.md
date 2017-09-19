---
RFC: unassigned
Title: Multiple Policyfiles and Teams - New Proposal
Author: Jon Cowie <jcowie@chef.io>
Status: Draft
Type: Standards Track
Replaces: RFC075
---

# Multiple Policyfiles and Teams - New Proposal

This RFC proposal aims to replace RFC075 (Multiple Policyfiles and Teams) with a new RFC which solves some of the blocking issues encountered with the original RFC.

The underlying motivation of this proposal will be identical to RFC075 (and full credit goes to Noah Kantrowitz for those), but the proposed implementation and functionality will differ significantly.


Policyfiles allow powerful new workflows around cookbook management, but one
area they are currently lacking is in dealing with multi-team organizations.

Currently each node can be attached to exactly one policy and one policy group.
The organizational manifestation of this is generally that there needs to be a
single "owner" for the policy of that node. This might be a CI tool like Jenkins
or Chef Delivery, but in a lot of cases in the current workflow this will be the
team within the company that uses the machine most.

This works great for the "owner" team, they can use all the power and
flexibility of the Policyfile workflow to its fullest. For other groups
managing software on the node, the picture is less clear. As an example, take a
database server owned by an application team. On this server we have a mix of
software, the MySQL install and configuration is manage by the DBA team while
ntpd and rsyslog are managed by a core infrastructure team. These live in
different cookbooks so for the most part things are copacetic. When the DBA
team wants to roll out a new MySQL configuration entry they update the cookbook,
recompile the policyfile, `chef push` that compiled policy to the QA policy
group, and then follow the usual Policyfile roll-out procedure. For the core
team, things are more complex. They could follow the same process, but this
would require careful coordination with the DBA team to ensure they don't have
a roll-out in progress (see https://yolover.poise.io/#21). They could also
release their cookbook changes either to a git branch or Supermarket server,
and then notify the DBA team that they should run a roll-out. Neither of these
are wonderful options.

## Motivation

   As a maintainer of core, company-level cookbooks,
    I want to release cookbook updates,
    so that I can deploy changes quickly.

With that said, a more specific case that isn't currently handled well:

The Database team owns MySQL and uses Chef to deploy it along with related configs
and tools. The Monitoring team owns Collectd, which is deployed to every server
in the organization, and they are responsible for configuring collectd and deploying
new versions of it as needed. The core team owns ntpd and is responsible for its
configuration.

All teams want to follow the snapshot-based approach where they have a mono-repo
of their own cookbooks with a few shared cookbooks pulled in via dependencies.
When a team wants to do a release they run `chef update` to recompile their
policy, and then use `chef push` to roll it out to each environment in sequence
(with time in-between for testing et al). The teams also do not want to have to
ask permission from any other team when deploying new versions of a service they
are responsible for.

This combination of a snapshot-based workflow without explicit team coordination
is not currently an easy thing to do with the policy systems.


## Specification

The proposed solution to the motivation described above is to allow a more composable form of policy which would be able to optionally include other policies inside it.

This could be done with the addition of a ```include_policy``` directive, an example of which is shown below:

```ruby
include_policy "base", git: "github.com/myorg/policies.git"
```

FOR DISCUSSION:

As far as I see it there are two possible ways this could be implemented:

* When a policy is uploaded, in addition to its lockfile being computed, any policies which include it are recompiled to pull in the lockfile for the changed policy, remaining otherwise unchanged.
* When a policy is uploaded, in addition to its lockfile being computed, any policies which include it are recompiled from scratch, pulling in updates to all changed lockfiles

Either way, the eventual intent would be that you would end up with the overall policy consisting of a "fused" collection of all included Lockfiles.

Another factor to consider here would be whether or not the contents of these various component policyfiles would be deep merged or have any form of precedence applied to them etc. Ie, how to we handle runlist and attribute clashes, or dependency clashes. 

## Downstream Impact

This solution would ideally not affect any tools which use existing policyfile behaviour, as the use of a single policyfile and all API calls and tools surrounding it would be unaffected - the changes here would be to how Policyfiles are compiled and updated.

## Copyright

This work is in the public domain. In jurisdictions that do not allow for this,
this work is available under CC0. To the extent possible under law, the person
who associated CC0 with this work has waived all copyright and related or
neighboring rights to this work.
