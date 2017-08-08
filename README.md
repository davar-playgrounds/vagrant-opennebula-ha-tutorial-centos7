-----------
Description
-----------

An example of a highly-available, hyper-converged `opennebula` setup https://docs.opennebula.org/5.4/advanced_components/ha/.
The setup provides high-availability of the `opennebula`/`gluster` `nodes` and live migration of the virtual machines.
Each `node` is an all-in-one `opennebula` `frontend`/`node`, with `glusterfs` shared datastore across the `nodes`.
For simplicity, each `gluster` `node` contains a single `brick`, and the number of `gluster` replicas equals the number of `opennebula`/`gluster` `nodes`.

`vagrant` is used to provide the infrastructure (OS installation and networking setup), and `ansible` playbooks
are used to configure `glusterfs` and `opennebula`, and `prometheus` is used for monitoring.
The `ansible` modules should be usable also on a physical host infrastructure providing the `nodes`
are resolving in DNS, contain a bridge network interface, and a block device to be used for `glusterfs`.

The `vagrant` test setup requires 1GB RAM per `node` and 20GB of additional storage space per `node` on the `vagrant` host under `/var/lib/libvirt/images`.
The Raft consensus algorithm used by `opennebula` requires at least 3 `nodes`.

Tested on:

- Ubuntu 16.04 x86_64, Vagrant 1.9.5


----------------------
Configuration overview
----------------------

`vagrant` reserves eth0 (the first interface) and this cannot be currently changed (see https://github.com/mitchellh/vagrant/issues/2093).

The `opennebula` bridge br1 is associated with eth1 interface. The eth2 interface is used for `glusterfs` communication.
The `opennebula`\`s web-interface `sunstone` is available on the the Virtual IP provided by Raft.
This Virtual IP is then used as the backend for an `apache` reverse proxy running on the management server port 80.
`vagrant` port forwarding feature is used to expose the `apache` port 80 on the management server to the `vagrant` host port 10080.



                                                                                                                               Virtual
                                                     ---------------                                                           Machines
               Management                            |             | eth0 192.168.122.0/24       -----------------------
                 server                              |   Vagrant   |-----------------------------| one1                |      ---------
                                                     |    HOST     |                             |                     |  e*  |       |
            ----------------- eth0 192.168.122.0/24  |             | br1(eth1) 192.168.123.11/24 | glusterfs-server    |------| one-0 |
            | mgt1          |------------------------|             |-----------------------------| opennebula-server   |      |       |
            |               |                        |             |                             | opennebula-sunstone |      ---------
            | ansible       | eth1 192.168.123.31/24 |             | eth2 192.168.124.11/24      | opennebula-node-kvm |
            | prometheus    |------------------------|             |-----------------------------|                     |      ---------
            | apache        |                        |             |                             | node_exporter       |  e*  |       |
            -----------------                        |             |                             | gluster_exporter    |------| one-1 |
                    |                                |             |                             | libvirt_exporter    |      |       |
                    | 192.168.123.10/24              |             |                             | opennebula_exporter |      ---------
                    |                                |             |                             -----------------------
                Virtual IP                           |             |                                                            .....
                                                     |             |
                                                     |             |
                                                     |             | eth0 192.168.122.0/24       -----------------------
                                                     |             |-----------------------------| oneX                |      ---------
                                                     |             |                             |                     |  e*  |       |
                                                     |             | br1(eth1) 192.168.123.1X/24 | glusterfs-server    |------| one-2 |
                                                     |             |-----------------------------| opennebula-server   |      |       |
                                                     |             |                             | opennebula-sunstone |      ---------
                                                     |             | eth2 192.168.124.1X/24      | opennebula-node-kvm |
                                                     |             |-----------------------------|                     |        .....
                                                     |             |                             | node_exporter       |
                                                     ---------------                             | gluster_exporter    |
                                                                                                 | libvirt_exporter    |
                                                                                                 | opennebula_exporter |
                                                                                                 -----------------------


------------------
Vagrant Host setup
------------------

Install `vagrant` https://www.vagrantup.com/downloads.html

Make sure nested virtualization is enabled on your host (e.g. your laptop):

        $ modinfo kvm_intel | grep -i nested

Install, configure libvirt:

- Debian/Ubuntu:

        $ sudo apt-get -y install kvm libvirt-bin
        $ sudo adduser $USER libvirtd  # logout and login again

  Install https://github.com/pradels/vagrant-libvirt plugin dependencies:

        $ sudo apt-get install -y ruby-libvirt
        $ sudo apt-get install -y libxslt-dev libxml2-dev libvirt-dev zlib1g-dev

  Disable the default libvirt network:

        $ virsh net-autostart default --disable
        $ sudo service libvirt-bin restart

- Fedora/RHEL7:

  TODO


------------
Sample Usage
------------

Install the required `vagrant` plugins and bring up the VMs:

        $ git clone https://github.com/marcindulak/vagrant-opennebula-ha-tutorial-centos7.git
        $ cd vagrant-opennebula-ha-tutorial-centos7
        $ for net in `virsh -q net-list --all | grep vagrant-opennebula-ha | awk '{print $1}'`; do virsh net-destroy $net; virsh net-undefine $net; done  # cleanup any leftover networks if this is not the first run
        $ vagrant plugin install landrush
        $ vagrant plugin install vagrant-libvirt
        $ vagrant up --no-parallel one1.mydomain one2.mydomain one3.mydomain mgt1.mydomain || :

After the VMs are up, `ansible` playbooks are used to configure `gluster` and `opennebula`. Due to common failures when downloading software from the internet, the playbooks will need to be executed until no errors are printed.

1. configure replicated `gluster` volume with count 3, used as the shared storage for `opennebula`

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-glusterfs-setup.yml" || :
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-glusterfs-setup.yml" || :

2. configure 3 independent `openenbula` frontends. They will later used to form the highly-available (HA) cluster.

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-setup.yml" || :
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-setup.yml" || :

3. on each node configure the `opennebula` `node`:

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-node-setup.yml" || :
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-node-setup.yml" || :

4. configure HA frontend leader:

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-ha-leader-setup.yml --extra-vars 'opennebula_ha_leader=one1.mydomain'" || :

5. configure the HA frontend followers:

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-ha-follower-setup.yml --extra-vars 'opennebula_ha_follower=one2.mydomain'" || :
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-ha-follower-setup.yml --extra-vars 'opennebula_ha_follower=one3.mydomain'" || :

The recent documentation of `opennebula` http://docs.opennebula.org/5.4/operation/ focuses on actions performed using the sunstone web-interface. sunstone port forwared by `vagrant` from the guest one1.myadmin:9869 (the current frontend leader) to the `vagrant` host port 19869 is accessible with credentials (defined in Vagrantfile) `onadmin`/`password`:

        $ firefox 127.0.0.1:19869

The older `opennebula` documentation http://docs.opennebula.org/4.14/design_and_installation/quick_starts/qs_centos7_kvm.html contains instructions how to start a VMs on the command line. All steps are to be performed on the frontend leader (currently one1.mydomain), and will be replicated over to `sqlite` instances on the frontend followers.

- add the `opennebula` worker `nodes` to the cluster:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onehost create one1.mydomain -i kvm -v kvm && onehost create one2.mydomain -i kvm -v kvm && onehost create one3.mydomain -i kvm -v kvm'"
        $ sleep 10
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onehost list'"

- update the system and default image datastores to use `gluster`:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onedatastore list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'echo NAME = system > system.one&& echo TM_MAD = shared >> system.one&& echo TYPE = SYSTEM_DS >> system.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onedatastore update system system.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onedatastore list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'echo NAME = default > default.one&& echo DS_MAD = fs >> default.one&& echo TM_MAD = shared >> default.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onedatastore update default default.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onedatastore list'"

- create a network template, consisting of potentially three virtual machines (SIZE = 3):

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevnet list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'echo NAME = private > private.one&& echo VN_MAD = dummy >> private.one&& echo BRIDGE = br1 >> private.one&& echo AR = [TYPE = IP4, IP = 192.168.123.100, SIZE = 3] >> private.one&& echo DNS = 192.168.123.1 >> private.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevnet create private.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevnet list'"

- fetch the CentOS 7 image the from OpenNebula's marketplace.

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'oneimage list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'oneimage create --name centos7 --path http://marketplace.opennebula.systems/appliance/4e3b2788-d174-4151-b026-94bb0b987cbb/download --datastore default --prefix vd --driver qcow2'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'oneimage list'"

  It takes some time to download this image, but one can proceed with further `opennebula` commands.

- create a VM template using that image:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate create --name centos7 --cpu 1 --vcpu 1 --memory 256 --arch x86_64 --disk centos7 --nic private --vnc --ssh --net_context'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate list'"

- update the template context with the root password (see https://docs.opennebula.org/5.4/operation/vm_setup/kvm.html). Specify `NETWORK=YES` (see https://docs.opennebula.org/5.4/operation/network_management/manage_vnets.html), otherwise opennebula won't configure network on centos7 VM (only lo interface will be present):

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'echo CONTEXT = [ USERNAME = root, PASSWORD = password, NETWORK = YES  ] > context.one'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate update centos7 -a context.one'"

- start the VM using this template:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate instantiate centos7'"
        $ sleep 60
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list'"

- collect the information about the host, network, image, template and VM (note that centos7 VM is visible as one-0 on one3.mydomain to virsh):

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onezone show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onehost show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevnet show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'oneimage show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onetemplate show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm show 0'"
        $ vagrant ssh one3.mydomain -c "sudo su - -c 'virsh dumpxml one-0'"

- verify one can ssh to the VM:

        $ vagrant ssh one1.mydomain -c "sudo su - -c 'yum -y install sshpass'"
        $ vagrant ssh one1.mydomain -c "sshpass -p password ssh -o StrictHostKeyChecking=no root@192.168.123.100 '/sbin/ifconfig'"

- test live-migrate the VM from one3.mydomain to one2.mydomain `node`:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list | grep one3'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm migrate 0 one2.mydomain --live'"
        $ sleep 10
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list | grep one2'"

Test a scenario when one1.mydomain `node` (the current frontend leader) is restarted due to maintenance. After the `node`
goes down it is expected that `opennebula` Virtual IP will be moved to another `node`, and the `gluster` volume continue to operate
with only two replicas. After the `node` is brought back `gluster` should recreate the 3 replica volume by synchronizing the updates to the `brick` on one1.mydomain,
and `opennebula` HA algorithm should join the one1.mydomain `node` as the second follower.

        $ vagrant suspend one1.mydomain
        $ vagrant ssh one2.mydomain -c "sudo su - oneadmin -c 'onezone show 0'"  # this will block until RAFT achieves consensus

At that point sunstone should be available from the current frontend leader (`X`) port 9869, forwared by `vagrant` as `X`9869 on the `vagrant` host:

        $ firefox 127.0.0.1:X9869

Let's migrate the VM back to one3.mydomain. The migration is initiated from the current leader, but using CLI it can be also performed from the follower:

        $ CURRENT_LEADER=`vagrant ssh one2.mydomain -c "sudo su - oneadmin -c 'onezone show 0'" | grep leader | awk '{print $2}'`
        $ vagrant ssh $CURRENT_LEADER -c "sudo su - oneadmin -c 'onevm migrate 0 one3.mydomain --live'"
        $ sleep 10
        $ vagrant ssh $CURRENT_LEADER -c "sudo su - oneadmin -c 'onevm list | grep one3'"

After bringing back one1.mydomain verify `gluster` synchonizes the volume (prometheus MDTMP) and RAFT sets the `node` as a follower:

        $ vagrant reload one1.mydomain
        $ vagrant ssh $CURRENT_LEADER -c "sudo su - oneadmin -c 'onezone show 0'"

Now let's simulate the case when the `node` which runs the centos7 VM, crashes, (automaticaly migrate the VM MDTMP) and the OS needs to be reinstalled (or a similar case of a server hardware update):

        $ vagrant destroy -f one3.mydomain
        $ sleep 180  # changing XMLRPC_TIMEOUT_MS may help to achive consensus faster, see https://forum.opennebula.org/t/frontend-ha-raft-problems/4702/22
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onezone show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list | grep one3'"  # onevm shows incorrectly that the VM is still running

Bring up the crashed `node`, and clear its old ssh server keys:

        $ vagrant up one3.mydomain
        $ vagrant ssh mgt1.mydomain -c "ssh-keygen -R one3.mydomain"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'ssh-keygen -R one3.mydomain'"
        $ vagrant ssh one2.mydomain -c "sudo su - oneadmin -c 'ssh-keygen -R one3.mydomain'"
        $ vagrant ssh mgt1.mydomain -c "ssh -o StrictHostKeyChecking=no one3.mydomain hostname"

Restore the `gluster` volume on the new `node`, using the information present on the running peer:

        $ UUID=`vagrant ssh one1.mydomain -c "sudo su - -c 'grep glusterfs-one3.mydomain -r /var/lib/glusterd/peers | cut -d: -f1 | cut -d/ -f6'"`
        $ VOLUME_ID=`vagrant ssh one1.mydomain -c "sudo su - -c 'getfattr -n trusted.glusterfs.volume-id /data/glusterfs/datastore1/brick1/brick 2>/dev/null | grep volume-id | cut -d= -f2-'"`
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-glusterfs-replace.yml --extra-vars 'gluster_server_node=one1.mydomain gluster_replace_node=one3.mydomain gluster_replace_uuid=$UUID gluster_volume_id=$VOLUME_ID'" || :

Verify `gluster` has synchonized the data to the brick on the new `node`:

        $ vagrant ssh one3.mydomain -c "df -h /data/glusterfs/datastore1/brick1'"

and configure `opennebula` frontend and `node` on the new server:

        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-frontend-setup.yml" || :
        $ vagrant ssh mgt1.mydomain -c "ansible-playbook -i /vagrant/ansible/hosts.yml /vagrant/ansible/playbook-one-node-setup.yml" || :

At that point there should be one leader and two followers, and the centos7 VM instance in the poweroff state:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onezone show 0'"
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list | grep one3 | grep poff'"

Resume the centos7 VM instance and verify it is accesible:

        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm resume 0'"
        $ sleep 30
        $ vagrant ssh one1.mydomain -c "sudo su - oneadmin -c 'onevm list | grep one3'"
        $ vagrant ssh one1.mydomain -c "sshpass -p password ssh -o StrictHostKeyChecking=no root@192.168.123.100 '/sbin/ifconfig'"


------------
Dependencies
------------

https://github.com/vagrant-landrush/landrush
https://github.com/vagrant-libvirt/vagrant-libvirt


-------
License
-------

BSD 2-clause


----
Todo
----

1. configure prometheus with: node_exporter, gluster_exporter, libvirt_exporter, opennebula_exporter

2. configure VM high-availability


--------
Problems
--------