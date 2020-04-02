Ansible Playbook for Patroni on Debian
======================================

This Ansible playbook allows to deploy a Patroni cluster.

It is an enhanced fork of [credativ's playbook](https://github.com/credativ/ansible-playbook-patroni-debian).
Please consult the [commit log](https://github.com/sourcepole/ansible-playbook-patroni-debian/commits/master)
to see the changes.

There's a [howto article](https://www.credativ.com/blog/integrating-patroni-debian)
on how to use this playbook.

This playbook is using the `patroni` packages provided by Debian and/or
PostgreSQL's `apt.postgresql.org` repository.

Those packages have been integrated with Debian's `postgresql-common`
framework and will look very similar to a regular stand-alone PostgreSQL
install on Debian. This is done by using `pg_createconfig_patroni` creating
setting the `manual` flag in the `/etc/postgresql/$PG_VERSION/$CLUSTER_NAME/start.conf`
config file. That will cause the postgres init script/systemd unit not to
start that cluster automatically. Instead `patroni` will take care to start
and manage that cluster.

For production use it should be hardened to use Ansible Vault for secrets.

The default topology consists of one node (`master`) acting as the DCS
(Distributed Consensus Store) server, and three PostgreSQL/Patroni nodes
(`pg1`, `pg2` and `pg3`), see `inventory`.

DCS Server
----------

Patroni requires a DCS for leader election and as configuration store.
Supported DCS server are Etcd, Consul and Zookeeper. If e.g. an Etcd cluster is
already available, then it can be configured either by editing
`templates/dcs.yml` or (if that suffices) by setting the `dcs_server` Ansible
variable to its IP address.

Usage
-----

Assuming password-less SSH access to the four nodes is configured, the playbook
can be run as follows:

```
ansible-playbook -i inventory patroni.yml
```

Supported Versions
------------------

The playbooks have been tested on Debian 9 (stretch), testing (buster) and
unstable (sid), as well as Ubuntu LTS 18.04 (bionic). As the
`apt.postgresql.org` PostgreSQL (and Patroni) packages are used, all supported
PostgreSQL versions can be installed in principle.

Note that Consul is unsupported as DCS on Debian 9 (stretch).
 
Variables
---------

The following useful variables can be set: 

 * `dcs` (`etcd` (default), `consul` or `zookeeper`)
 * `dcs_server_ips` (default: undefined, if set will override `dcs_servers` - see below)
 * `postgresql_cluster_name` (default: test)
 * `postgresql_major_version` (default: 11)
 * `postgresql_data_dir_base` (default: `/var/lib/postgresql`)
 * `postgresql_network` (default: network of eth0 interface)
   network that access postgres and do queries. md5 auth is required.
 * `postgresql_repl_network` (default: network of eth0 interface)
   network that can do replication. md5 auth is required.
 * `patroni_replication_user` (default: `replicator`)
 * `patroni_replication_pass`
 * `patroni_postgres_pass`
 * `vip`
 * `use_certificates` (only applies to `etcd` DCS, default: false, see below)

If `dcs_server_ips` is set, then it will be used. If not set, then the IPs of
hosts of the `dcs_servers` inventory group will be used.

Example: `ansible-playbook -i inventory -e dcs=consul patroni.yml`

For further configuration and understanding it might be useful to consult
the [patroni configuration settings documentation](https://github.com/zalando/patroni/blob/master/docs/SETTINGS.rst).
It might be necessary to modify the [patroni template accordingly](templates/config.yml.in).

Also please see https://github.com/zalando/patroni/issues/878#issuecomment-441558744 to
understand how patroni settings propagate.

Running multiple instances of Patroni/PostgreSQL
------------------------------------------------

By default, Patroni uses 8008 as the API REST port. The automatic Patroni
configuration generation will increment this for every additional cluster, so
running

```
ansible-playbook -i inventory -e postgresql_cluster_name=test2 --tags=config pgsql-server.yml
```

will result in a second cluster, `11/test2` using PostgreSQL port 5433 and
Patroni API port 8009.

Managing a virtual IP address with `vip-manager`
------------------------------------------------

The `vip-manager` package allows to expose a virtual IP address (VIP) for the
leader node by monitoring the leader key in the DCS and setting or removing the
configured VIP for the local node depending on leader status.

When the `vip` variable is set to an IP address, an appropriate configuration
file for `vip-manager` will be written and the cluster-specific `vip-manager`
service will be started.

Etcd with certificate based authentication
------------------------------------------

Configuring etcd to use certificate based, authenticated connections is
supported. Please set:

    use_certificates: true

(see `patroni.yml` for details)

You need to:

* either use externally generated certificates or
* or you can let this playbook generate a CA and certificates

If you want to let this playbook generate the CA and certificates, then
you need to:

* uncomment the line:
  
      - import_playbook: certificates-sp.yml

  in `patroni.yml`

* configure `certificates-sp.yml`

* install the required `tpo.local_certificate_authority` role like this:

      ansible-galaxy install tpo.local_certificate_authority

Rewinding/Recloning outdated former primaries
---------------------------------------------

If a failover has occured and the old leader has additional transactions that
do not allow for a clean change of timeline, `pg_rewind` can be  used in order
to rewind the old leader so that it can reinitiate streaming replication to the
new leader.  For this to happen, the variable `patroni_postgres_pass` needs to
be set in `vars.yml`.  If this is not the case, a full reclone will be done.

In both cases, the former primary's data directory will be gone, so if you
prefer to have the primary stay down for manual inspection instead, you should
comment out (or set to `false`) the parameters `use_pg_rewind`,
`remove_data_directory_on_rewind_failure` and
`remove_data_directory_on_diverged_timelines`.

