============
Poutacluster
============

**NOTE: This is a work in progress**

Pouta-virtualcluster is a helper script and a set of Ansible playbooks to quickly setup a cluster in **pouta.csc.fi**
IaaS service. It draws heavily from *ElastiCluster* by *Grid Computing Competence Center, University of Zurich*.
Most of the Ansible playbooks are based on ElastiCluster, however provisioning the VMs is written from scratch to
support volumes (persistent storage) and runs directly against OpenStack Python API.

Currently poutacluster can provision:

* Basic cluster infra on top of CentOS 6.5 with one frontend and N compute nodes

  - frontend and compute nodes can have different images, flavors and keys
  - frontend has a public IP
  - there is a volume for local persistent data
  - frontend has a separate shared volume, exported with NFS to the worker nodes from */shared_data*
  - frontend also exports */home*

* Ganglia for monitoring
* GridEngine for batch processing
* Apache Hadoop 1.2.1
* Apache Spark 1.0.2

How it works
============

There are two separate parts:

- provisioning the VMs and related resources
- configuring the VMs with Ansible

Provisioning
------------

Provisioning VMs for cluster goes roughly like this:

* The cluster configuration is read from YAML file provided by the user.
* The current state of provisioned resources is loaded using OpenStack APIs from pouta.csc.fi
* Missing VMs are provisioned

  - first frontend, then appropriate number of nodes
  - naming: *[cluster-name]-fe* and *[cluster-name]-node[number]*. If your cluster name was be *my-cluster*,
    you would get

      + my-cluster-fe
      + my-cluster-node01
      + my-cluster-node02
      + ...

  - VMs are launched from the specified image with specified flavor
  - volumes are created or reused and attached
  - template security groups are created if these don't exist already


* Ansible host inventory file is created, mapping VMs to assigned roles

Shutdown is done in reverse order, starting with the last nodes and finally shutting down frontend. Only after the cluster
has been shut down, you can wipe the persistent storage for the cluster.

See the example below.

Configuration
-------------

There is a collection of Ansible playbooks in *ansible/playbooks* directory, all collected to *ansible/playbooks/site.yml*.
Based on the role/group assignment in Ansible inventory (which in turn is generated from provisioning state and cluster
configuration in cluster.yml by poutacluster script) a set of tasks is launched on each VM.

The playbooks are designed to be idempotent, so that you should be able to run them at any time, and they will only make
changes in the configuration if necessary. Also, if you change the number of nodes in the cluster, playbooks can be
re-applied to reflect the change.

Prerequisites
=============

Getting started with pouta.csc.fi
---------------------------------

To use CSC Pouta cloud environment you will need

* credentials for Pouta
* basic knowledge of Pouta and OpenStack

See https://research.csc.fi/pouta-user-guide for details

Setting up bastion host
-----------------------

Note: Here we assume that you are already past the basic steps mentioned above.

Create a small management VM to act as a "bastion" host (http://en.wikipedia.org/wiki/Bastion_host)

* Log into https://pouta.csc.fi
* If you are member of multiple projects, select the desired one from the drop down list on top left
* Create a new security group called, for example, 'bastion'

  - go to *Access and Security -> Security groups -> Create Security Group*
  - add rules to allow ssh for yourself and other admins
  - normal users do not need to access this hosts
  - keep the access list as small as possible to minimize exposure

* Create an access key if you don't already have one

  - go to *Access and Security -> Keypairs -> Create/Import Keypair*

* Boot a new VM from the latest CentOS 6 image that is provided by CSC

  - go to *Instances -> Launch Instance*
  - Image: Latest public Centos image (CentOS 6.5 at the time of writing)
  - Flavor: tiny
  - Keypair: select your key
  - Security Groups: select only *bastion*
  - Network: select the desired network (you probably only have one, which is the default and ok)
  - Launch

* Associate a floating IP (allocate one for the project if you don't already have a spare)

* Log in to the bastion host with ssh as either *cloud-user* or *ubuntu* user, depending on the image::

    ssh cloud-user@86.50.168.XXX:
    
* Run commands requiring superuser privileges with sudo or switch to root user with:: 
    
    sudo -i

* update the system and reboot to bring the host up to date::

    yum update -y && reboot

* install EPEL repo, openssh-clients, bash-completion, git, Python yaml-support, Ansible and OpenStack clients::

    rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    yum install -y bash-completion openssh-clients python-novaclient python-cinderclient git python-yaml ansible

* import your OpenStack command line access configuration

  - see https://research.csc.fi/pouta-credentials how to export the openrc
  - use scp to copy the file to bastion from your workstation::

    [me@workstation]$ scp openrc.sh cloud-user@86.50.168.XXX:

* test the clients (enter your Pouta password when asked for)::

    source openrc.sh

    nova image-list

* create a new key for cluster access (keeping bastion access and cluster access separate is a good practice)::

    ssh-keygen

* import the key::

    nova keypair-add  --pub-key .ssh/id_rsa.pub cluster-key


Installation
============

Next we install *poutacluster* on the bastion host::

    yum install ansible -y
    cd
    git clone https://github.com/CSC-IT-Center-for-Science/pouta-virtualcluster
    mkdir ~/bin
    ln -s ~/pouta-virtualcluster/python/poutacluster.py ~/bin/poutacluster
    ln -s ~/pouta-virtualcluster/ansible ~/ansible
    cp ~/pouta-virtualcluster/ansible/cfg/ansible-centos6.cfg ~/.ansible.cfg

Now *poutacluster -h* should give you basic usage. See examples below for more details.

Examples
========

Cluster life-cycle walk-through
-------------------------------

Log in to the bastion host, source the openrc.sh and start deploying the cluster:

* create a new subdirectory for the cluster configuration in your home directory::

    mkdir ~/my-cluster
    cd ~/my-cluster

* copy *cluster.yml.template* to *~/my-cluster/cluster.yml* and open it for editing::

    cp ~/pouta-virtualcluster/cluster.yml.template cluster.yml
    vi cluster.yml

* you can also edit the definition on your workstation and then copy it over to the bastion. The template can
be found at https://github.com/CSC-IT-Center-for-Science/pouta-virtualcluster

* check, edit or fill in:

  - cluster name
  - ssh-key name
  - public IP
  - image
  - flavors
  - volume sizes (NOTE: when testing, keep the volume size small, otherwise deleting the cluster storage will take a long time). Keep the volume names and order as they are.
  - groups - you can comment out software groups that you don't need

* bring the cluster up with a frontend and two nodes::

    poutacluster up 2

* check what *info* shows about the state::

    poutacluster info

* ssh in to the the frontend and test the cluster

* check the web interfaces for Ganglia, Hadoop and Spark. Urls are printed out at the end of the run

* try resetting the nodes::

    poutacluster reset_nodes

* bring the cluster down to save credits (permanent data on volumes is still preserved)::

    poutacluster down

* bring the cluster up again, this time with 4 nodes::

    poutacluster up 4

* destroy the cluster by first bringing it down and then getting rid of the volumes::

    poutacluster down
    poutacluster destroy_volumes

General cluster
---------------
Check uptime on all the hosts on cluster frontend::

    pdsh -w mycluster-node[01-04] uptime

Reboot the nodes::

    pdsh -w mycluster-node[01-04] reboot

Add a user and test NFS::

    useradd -u 1010 bill
    passwd bill
    pdsh -w mycluster-node[01-04] useradd -u 1010 --no-create-home bill
    su - bill
    ssh mycluster-node01 touch hello-from-node01
    ls
    exit


GridEngine
----------

As a normal user (or centos), test job submission::

    cd
    for i in {001..016}; do qsub -b y -N uname-$i uname -a; done
    cat uname-0*.o*

The jobs are probably executed on different nodes.

Create a few empty 1G files on the NFS share and calculate sha256 sums over zero data::

    sudo mkdir /shared_data/tmp
    sudo chmod 1777 /shared_data/tmp
    for i in {001..050}; do truncate --size 1G /shared_data/tmp/zeroes.1G.$i; done
    for i in {001..050}; do qsub -b y -N shasum-$i sha256sum /shared_data/tmp/zeroes.1G.$i; done
    cat shasum-*.o*

During the test, you should see quite a lot of network traffic from frontend out to the nodes, as the sparse files are
read and NFS is feeding a lot of zeroes to the sha256sum -processes on the nodes. You can open another terminal (or use
a multiplexer like *tmux* or *screen*) and run *dstat -taf 10* for some real time monitoring on the frontend.

Hadoop
------

Running terasort with 100GB dataset. Make sure you have big enough *shared_data* and *local_data* -volumes provisioned.::

    # generate data (with 8 'small' nodes, this should take around 6 minutes)
    # map tasks tuned to match the size of the cluster (8 small nodes, 4 cores each)

    hadoop jar /usr/share/hadoop/hadoop-examples-1.2.1.jar teragen -Dmapred.map.tasks=32 1000000000 /user/hduser/terasort-input

    # sort (with 8 'small' nodes, this should take around 15 minutes)
    # reduce tasks tuned to match the size of the cluster (8 small nodes, 4 cores each)

    hadoop jar /usr/share/hadoop/hadoop-examples-1.2.1.jar terasort -Dmapred.reduce.tasks=32 /user/hduser/terasort-input /user/hduser/terasort-output


Some useful admin commands::

    # get status report for hdfs (HADOOP_USER_NAME is needed for admin access)

    HADOOP_USER_NAME=hdfs hadoop dfsadmin -report

    # balancing the HDFS data across nodes: set the balancer bandwidth to 100MB/sec and run balancer
    HADOOP_USER_NAME=hdfs hadoop dfsadmin -setBalancerBandwidth 100000000
    HADOOP_USER_NAME=hdfs hadoop balancer -threshold 1

    # check HDFS
    HADOOP_USER_NAME=hdfs hadoop fsck /

    # list running jobs
    hadoop job -list

Spark
-----
Word count example with a random 6MB file found in the internet containing text. The file is concatenated 10000 times,
resulting 61GB of text data. Make sure you have big enough *shared_data* and *local_data* -volumes provisioned. Another
good source of big text is Wikipedia database dumps (http://en.wikipedia.org/wiki/Wikipedia:Database_download).

First download some ascii text and concatenate it to NFS shared directory::

    sudo mkdir /shared_data/tmp
    sudo chmod 1777 /shared_data/tmp
    cd /shared_data/tmp
    wget http://norvig.com/big.txt
    for i in {1..10000}; do cat big.txt >> big.txt.x10000; done

Then upload it to HDFS also (this will take some time)::

    hadoop dfs -put big.txt.x10000 /sparktest/big.txt.x10000

Make sure Spark is running::

    sudo /opt/spark/sbin/start-all.sh

Start a Spark shell with 8GB worker nodes in the cluster::

    /opt/spark/bin/spark-shell --master spark://mycluster-fe:7077 --executor-memory 8G

Note that logs will be printed to the shell and it might look like the prompt is not ready. Hit *Enter* a few times to
get the *scala>* -prompt.

First we can test reading the input from NFS and writing the results to HDFS::

    val bigfile = sc.textFile("file:///shared_data/tmp/big.txt.x10000")
    val counts = bigfile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
    counts.saveAsTextFile("hdfs://mycluster-fe:9000/sparktest/output-1")

Note: Spark is lazy in evaluating the expressions, so no processing will be done before the last line.

Then test HDFS to HDFS::

    val bigfile = sc.textFile("hdfs://mycluster-fe:9000/sparktest/big.txt.x10000")
    val counts = bigfile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
    counts.saveAsTextFile("hdfs://mycluster-fe:9000/sparktest/output-2")

Probably these hadoop dfs -commands will be handy, too::

    hadoop dfs -ls /sparktest
    hadoop dfs -du /sparktest/*
    hadoop dfs -rmr /sparktest/output-1-gone-wrong

Missing bits
============

* support for Ubuntu

* online resize

* persistent home directory

* HDFS resize has to be done manually when scaling down

* Spark does not start automatically after a reboot. To start it run::

    [root@fe /root]# /opt/spark/sbin/start-all.sh


