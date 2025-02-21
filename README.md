## Quickstart

- Requires a recent version of Ansible
- Expects Debian 8, Ubuntu 16.04, or CentOS 7 or newer (depending on packaging)

These playbooks deploy a full Opencast cluster, from packages.  To use them, first edit the `hosts` inventory file
to contain the hostnames of the machines on which you want Opencast deployed, and edit `group_vars/all.yml` to set any
parameters you might want.  Prior to running the playbook you need to install the dependency roles.  This can be done
with:

	ansible-galaxy install -r requirements.yml

Then run the playbook, like this:

	ansible-playbook -K -i hosts opencast.yml

When the playbook run completes, you should be able to see Opencast on port 80, on the target machines.

This playbook is *not* meant to deploy production machines as is, instead use it to form the basis of a playbook which
better matches your local needs and infrastructure.

## Detailed Docs

#### Hosts

To begin, we first must first change the hosts file to match our environment.  Each play (e.g: `[fileserver]`) can have
between zero and N hosts associated with it.  A given host can also be associated with multiple playbooks.  In [the 
default `hosts` file](hosts) we see that the `admin.example.org` host will play the role of the fileserver, mysql server,
and admin node.  We also see that the `ingest` playbook has no hosts associated - that means that it will not be used.
These scripts do not have a way to define dependencies between playbooks, which means (as an example) that you can
deploy a cluster without an admin node, which is an invalid configuration.  Keep this in mind when defining your host
environment!  At this point, please change the hosts to match your environment.

Host list requirements:

 - `fileserver`: Exactly 1 host.  With 2 or more hosts only the first will be used, with 0 hosts you will encounter errors.
 - `mariadb`: Exactly 1 host.  With 2 or more hosts only the first will be used, with 0 hosts you will encounter errors.
 - `activemq`: Exactly 1 host.  With 2 or more hosts only the first will be used, with 0 hosts you will encounter errors.
 - `allinone`, `admin`, `adminpresentation`: At most 1 host in all of these groups combined.  The admin 
   node used by the cluster is defined by the _first_ entry of the `admin_node` group.  This means that if you define an
   a host under `admin`, and a host under `adminpresentation` then the cluster will use the host under `admin`.
 - `allinone`, `presentation`, `adminpresentation`: At most 1 host in all of these groups combined.  The presentation
   node used by the cluster is defined by the __first__ entry of the `presentation_node` group.  This means that if you
   define a host under `presentation`, and a host under `adminpresentation` then the cluster will use the host under
   `presentation`.
 - `allinone`, `worker`: At least 1 host in at least one of these groups.  There are no outright errors if
   you deploy a cluster without any workers, but Opencast will be unable to process any recordings.  In general this is
   a useless configuration, unless you are strictly testing configuration files.

Host configuration for these scripts:
 - A shared user (by default named `ansible`) who accepts your SSH key, has the same password on all nodes in `hosts`,
   and has `sudo` access.

Configuration Hints:
 - The most common production configuration is a single admin, single presentation, and N workers.
 - Decide where you want your admin and presentation nodes first.  Once media has been processed, it's very difficult
   to change the admin and presentation hostnames.
 - Opencast uses the hostname as defined in the hosts file.  Use the externally resolvable name (ie, admin.example.org,
   rather than just admin), otherwise you will encounter problems in the future.

#### Variables

Most variables are exposed in the `group_vars/all.yml` and `group_vars/opencast.yml` files.  These files should be
well documented, so please take a look.

### Troubleshooting

 - __My install fails with an error complaining that the SEQUENCE table already exists__

You already have some data in the schema you are installing to.  This might be pre-existing Opencast data, so we fail
the install rather than potentially overwrite your data.  If you are sure you want to overwrite it, use the `reset` tag
after reading the caveat in the Advanced Usage section below.

 - __My install fails complaining that a package could not be installed__

Try again in a day or so.  If you are particularly unlucky you might have caught us updating the repository, or there
might be an issue with your distribution's mirroring systems.  If this does not resolve itself in a day please complain
on list!

#### Known issues for production use

 - These playbooks ship with the default upstream Opencast credentials.  These *must* be changed for production use.
 - ActiveMQ is left in a world-connectable state.  Setup a firewall between your ActiveMQ node and the untrusted network.
 - NFS is restricted to the hosts involved in the playbook, but does not have any other security in place and is running
   in version 3.
 - HTTPS configuration is possible (`oc_protocol` in `group_vars/opencast.yml`), however these playbooks do not cover
   setting up and issuing certificates.

### Ideas for Improvement

Here are some ideas for ways that these playbooks could be extended:

 - Enable Solr clustering for relevant services
 - Enable Elastic Search server clustering for relevant services
 - Add security to various installed services to prevent them from being world accessible (see above)
 - Multiple tenant support
 - Error checking for group settings.  You can define insane configurations such as multiple admin nodes, or clusters
   without ActiveMQ.

### Advanced Usage

The playbook contains tags, which cause Ansible to only execute part of the full playbook.  These tags are currently:

 - `config`, which updates all of the configuration files created by this playbook, then restarts Opencast.  Don't do
   this on your production system, figure out the correct settings on your test instance then reinstall for production.
 - `uninstall`, which removes the Opencast packages, but does not remove the data.  Note that this uninstalls the 
   current branch's packages.  If you have r/5.x checked out and uninstall it will uninstall the 5.x packages, but not
   the 4.x or 6.x.
 - **DANGER** `reset`, which removes _all_ Opencast user data, but leaves the packages installed.  This is only useful
   for testing situations, and *will happily wipe out your production data without further prompting.*

Hints:
 - Running `--tags config,reset` resets the database, filesystem, and ensures all of your config files are in the
   expected state, then restarts Opencast.
 - If you are switching between Opencast versions when testing, first run `--tags uninstall` to remove the packages,
   then checkout the correct branch of these playbooks.  Then run `--tags untagged,reset` to install and reset.
   Without the reset tag you will see an error when the database is imported because the schema will already exist.
