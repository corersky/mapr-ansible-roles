[![Build Status](http://test-1085049401.us-east-1.elb.amazonaws.com/buildStatus/icon?job=mapr-ec2-build)](http://test-1085049401.us-east-1.elb.amazonaws.com/job/mapr-ec2-build/)

IMPORTANT!
==

The AWS playbooks now assume the use of internal ec2 hostnames, in a VPC. This means the playbooks below should be run from an EC2 host in your VPC, or over a VPN connection. 


mapr-ansible-roles
=======================

Intro
======

This repo contains Ansible playbooks that do the following:

* Launch EC2 instances for MapR [playbooks/aws_bootstrap.yml](https://github.com/vicenteg/mapr-ansible-roles/blob/master/playbooks/aws_bootstrap.yml)
* Apply MapR OS prerequisites per http://doc.mapr.com/display/MapR/Preparing+Each+Node [playbooks/prerequisites.yml](https://github.com/vicenteg/mapr-ansible-roles/blob/master/playbooks/prerequisites.yml)
* Install a cluster [playbooks/install_cluster.yml](https://github.com/vicenteg/mapr-ansible-roles/blob/master/playbooks/install_cluster.yml)
* Install MapR Metrics & MySQL
* Install ecosystem projects:
  * Hive
  * Impala
  * HBase
* Print some information about the resulting cluster 
  * webserver URLs

This project also includes a Vagrantfile that creates a single local VM instance, or a local VM cluster, suitable to run MapR. The playbooks here can be used either for vagrant instances or EC2 instances.

This will install MapR release 4.0.1 by default. It will not work for earlier releases.

This does not include a license, so to enable licensed features, you'll need to obtain a license here: http://www.mapr.com/user/addcluster

AWS Instances
==============

AWS instances you create will be spot instances by default unless you comment out the lines in `aws_bootstrap.yml` that specify the bid price. If you comment those out, you will get on demand instances, which will cost significantly more. The recommendation is that you consider using spot instances if the following apply:

1. You will not use the cluster for a live demonstration in front of important people
2. You will not store any important or long-lived data
3. You are OK with the cluster being terminated (i.e., destroyed forever) without warning
4. You don't need the instances to survive a reboot

If any of the above are not true (i.e, you will be doing a live demo, or you need the cluster to come up if you reboot a node) you should use on demand instances.

Be sure to check the latest spot prices for the instances you're looking to create. Also keep in mind that not all instances are available as spot instances. Importantly, remember that spot instances can be terminated by Amazon at any time if the bid price goes above the maximum price you set. So don't use spot instances if you absolutely must keep the instances running!

If you prefer on-demand instances (so that you're not at risk of automatic termination), edit `playbooks/aws_bootstrap.yml` and comment out the lines with `spot_price`.

Prerequisites - Vagrant
=======================

You will want VirtualBox and Vagrant installed. The versions I'm using are below.

```
$ VirtualBox --help
Oracle VM VirtualBox Manager 4.3.8
```

```
$ vagrant --version
Vagrant 1.5.1
```

Prerequisites - AWS
====================

Please review [the README for mapr-aws-bootstrap](https://github.com/vicenteg/mapr-singlenode-vagrant/tree/master/playbooks/roles/mapr-aws-bootstrap).

Before starting the installation, you will want to edit some variables. Variables can be modified in the top-level playbooks. For each group, you can inspect the role variables. An example, so you know what to look for is here:

```
- hosts: CentOS;RedHat
  max_fail_percentage: 0
  sudo: yes
  roles:
    - { role: mapr-redhat-prereqs,
        mapr_user_pw: '$1$yoPLWBQ6$6fvQchDTBu3Ccs3PVURpA.',
        mapr_uid: 2147483632,
        mapr_gid: 2147483632,
        mapr_home: '/home/mapr',
        mapr_version: 'v4.0.1' }
```

Edit them in the file, or you can override these using the `--extra-vars` argument to ansible-playbook. For example, the argument would look like this to change mapr_version:

```
--extra-vars "mapr_version=v4.0.1"
```

You could also copy `playbooks/roles/mapr-aws-bootstrap/defaults/main.yml` to `playbooks/roles/mapr-aws-bootstrap/vars/main.yml`. Variables set in `vars/main.yml` override variable set in `defaults/main.yml`. But I think using the top level role variables or overriding them on the ansible-playbook command line is easier.

For mapr-metrics, review [its README file](https://github.com/vicenteg/mapr-ansible-roles/blob/master/playbooks/roles/mapr-metrics/README.md).


Pre-Install - AWS & Vagrant
===========================

1. Create a password for the mapr user. You'll use the mapr user to log into MCS. Use `openssl passwd -1` and put the hashed password in either extra-vars or in the role variables in the top-level playbook. 
2. Make sure that the list of disks you want to use aligns with the disks present on the systems. If you didn't change the bootstrap playbook, you should not have to do anything.
3. Check all the ec2 related variables. Chances are excellent you need to change something there.

Installation - Vagrant
=======================

Issue `vagrant up`, and watch as vagrant sets up your VM and provisions it.


Installation - AWS
===================

After modifying configuration files as needed, run the playbook as follows, being sure to substitute the path to your private key.

```
ansible-playbook -i playbooks/cluster.hosts --private-key <path/to/your/private_key> -u root \
	--extra-vars "mapr_cluster_name=my.cluster.com mapr_version=v4.0.1" \
	playbooks/install_cluster.yml
```


Post-Install - Vagrant - license key
=======================

After issuing `vagrant up`, the VM should be provisioned. Place your license key file in the directory along side the Vagrantfile. In your Vagrant directory, say:

`vagrant ssh`

And you should be dropped into a shell in your VM.

If your license key is called demolicense.txt, the steps following will add the key, start the NFS gateway and (additional) CLDB service. 

```
sudo maprcli license add -license /vagrant/demolicense.txt -is_file true
sudo maprcli node services -filter "[csvc==nfs]" -nfs start
```

No need to manually mount the loopback NFS - warden will take care of that for you.

Post-Install - AWS - license key
=======================

After the installation is complete, the ansible plays will print the webserver URLs for you. Copy and paste the URL into your browser, log in with user `mapr` and the password you set earlier. Then add the license key via upload to MCS, using the upper right hand "Manage Licenses" link. You could, of course, upload the file using scp and then add it as above (with maprcli) if you choose.

Once done, start up NFS and the additional CLDB instance(s):

```
sudo maprcli node services -filter "[csvc==nfs]" -nfs start
sudo maprcli node services -filter "[csvc==cldb]" -cldb start
```

No need to manually mount the loopback NFS - warden will take care of that for you.

Post-Install Validation - Vagrant and AWS
========================

For a simple smoke test, try running a teragen:

```
hadoop jar /opt/mapr/hadoop/hadoop-0.20.2/hadoop-0.20.2-dev-examples.jar \
	teragen 100000 /user/vagrant/teragen
```

If you don't see a java traceback, things are probably mostly OK.

For a little more, try running the test playbook:

```
ansible -i playbooks/cluster.hosts playbooks/test_cluster.yml
```

If nothing comes back failed, you should be ready to rock.


Have fun.

Potential Gotchas
==

## Amazon Credentials

If you see messages like the following:

```
msg: No handler was ready to authenticate. 1 handlers were checked. ['QuerySignatureV2AuthHandler'] Check your credentials
failed: [172.16.2.80 -> 127.0.0.1] => {"failed": true}
```

You missed the step about sourcing your Amazon credentials.
