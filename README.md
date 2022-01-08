# ansible-pull-example

This repo serves as an example of ansible's pull mode.

I found pull mode somewhat under-documented, so this repo is
intended to provide a practical example for people wishing to get
started with ansible-pull. In particular, it collects a bit of
utility code that should enable some basic workflows.

Running ansible in pull mode makes a different trade-off than the
usual centralized ansible workflow. The main benefits are the
implicit scaling to a large number of nodes, a simple
repository-oriented workflow, and avoiding the need for
awx/tower. Drawbacks are mainly that the pull workflow is
somewhat obscure, results in eventually consistent
infrastructure, and has some gotchas detailed below.

## invoking ansible

You'll want to invoke ansible like this if you use this ansible-pull setup:

```
# pull mode (suitable for automation)
$ ansible-pull -U https://git.example.com/ansible.git -i "$(hostname --short),"

# push mode (development)
$ ansible-playbook -i inventory ./playbook.yml --limit foo.example.com
```

## practical ansible-pull

ansible-pull changes the ansible workflow a little. Usually
ansible is run on a central server and targets a set of remote
hosts. In pull mode, each remote host pulls the whole ansible
repository from source control and runs a copy of ansible with
only itself as the sole "remote" host. This results in a few
oddities:

  1. `inventory_hostname` defaults to `localhost`
  2. groups are unavailable
  3. each play is only applied to the current host (`delegate_to` doesn't work)
  4. pull codebases are usually slower to iterate on when
     developing
  5. every host needs a copy of ansible and all modules and their dependencies used by the playbooks installed
  6. hosts must be able to pull the ansible repo
  7. credential management requires a separate solution

Some approaches for mitigating these oddities follow:

While the `inventory_hostname` is always localhost by default, it
can be explicitly specified when invoking `ansible-pull`.

The unavailability of groups is worked around by tagging each
host with their groups in `host_vars` instead of including this
grouping in an inventory. Playbooks can then use this mapping to
synthesize the equivalent push mode groups.

These sythetic groups are turned into proper groups by the
inventory script that I've provided. This enables push-style
development, which allows iterating on changes more quickly than
solely relying on the pull flow.

In pull mode, ansible calls a playbook named `local.yml` by
default; the `local.yml` in this repo does the group
synthesis that I described, and then goes on to invoke the
example playbook `playbook.yml`. When developing, you'd invoke
`playbook.yml` in push mode instead, using the inventory script
provided.

The one-host-per-play limitation doesn't really have a
workaround. If you rely on `host_vars` or facts from other hosts
in a play, you'll need to provide some other data plane for
sharing this information. Some reasonable solutions are static
info in `host_vars`, custom lookup plugins, or something like etcd.
However, consider that pull mode may not be the right solution if
your workflows rely on cross-host communication.

You'll want to install at least ansible on every host
participating in pull mode. Note that this also applies to the
dependencies listed on each module, and the modules themselves,
too, if they aren't in ansible base. In large sites this can add
up to a considerable amount of total disk space.

Requiring hosts to track the repository containing your playbooks
also implies a few things. The load on your repository server
scales linearly in the number of hosts using pull mode and
firewalling becomes more difficult. Unless your repository is
anonymously and globally readable, you'll need some way of
provisioning initial credentials on your target hosts to be able
to access it at all. SSH certificates may also be of interest
here.

This credentials issue also shows up elsewhere. In many setups,
the central server will have some level of access to secrets that
are then pushed to remote hosts. In pull mode, each remote host
is their own central server, so each one requires access to
secrets. This makes several solutions that work well in push
mode, like ansible vault, difficult to deploy securely in pull
mode. Larger setups will probably want to set up something like
Hashicorp's Vault or similar secret management services.

Finally, I've provided a sample `ansible-pull` role, some example
playbooks, and `host_vars` to help you get started.
