# ansible-pull-example

This repo serves as a example of ansible's pull mode.

I found documentation on pull-mode use relatively scarce, so this repo
is intended to both provide a practical skeleton project for people
wishing to get started with ansible-pull and collect some of the
utility code for pull-mode that I've found quite useful.

Some of the benefits are that the resultant system is loosely coupled,
implicitly scales, has a repo/CI-friendly workflow, and avoids the
need for awx/tower. However, the resultant system is both eventually
consistent and distributed, which brings challenges of its own.

## Practical ansible-pull

In pracice, ansible-pull changes the ansible workflow a little:
  1. hostname is always `localhost`
  2. groups are unavailable
  3. there's only the current host in each play / no `delegate_to`
  4. unifying pull/push codebases is difficult (but necessary for easy development)
  5. every host needs a copy of ansible installed
  6. hosts must be able to pull the ansible repo

(1) can be circumvented by explicitly specifying an inventory during
pull. (2) is worked around by tagging hosts with their groups in
host_vars, and then dynamically loading these. To solve (4), an
inventory script is provided that is aware of the group tagging from
(2).

(3) can't really be solved. If you rely on access to host_vars/facts
from other hosts in a play, you'll probably want to provide them some
other way. includes/host_vars/etcd are reasonable, but consider that
`ansible-pull` workflows may not be the right tool if you make heavy
use of these.

As stated in (4), you'll need to install ansible on the target
host. This also means that dependencies of ansible modules for both
the host and target side must be available on your host. You'll
probably also need to set up deploy keys or certificates to solve
(5). Take a look at ansible vault and `--vault-password-file`, too,

Finally, I've provided a sample ansible-pull role to help you get started.

## Invocation
  
You'll want to invoke ansible like this if you use this ansible-pull setup:

```
# pull mode (suitable for automation)
foo$ ansible-pull -U https://git.example.com/ansible \
       -i "$(hostname --short),"

# push mode (development)
$ ansible-playbook -i inventory ./playbook.yml --limit foo.example.com
```
