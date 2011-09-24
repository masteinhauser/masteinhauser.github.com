## How To: Install CoroSync + Pacemaker on CentOS 5.6
In this post we will be setting up CoroSync and Pacemaker on CentOS to create an easily managed cluster for different resources. There are many options available to create a cluster, Hearbeat being one of the oldest, but the development team I am a part of have settled on this combination as it is the most likely to receive security and feature updates as clustering systems change. It also takes advantage of some of the items that Heartbeat does a good job of managing currently while adding far easier management and scalability that wasn't possible with only Heartbeat 2.
<!--more-->

Install EPEL repository for packages that don't exist in vanilla Redhat/CentOS.
``` bash
su -
rpm -Uvh http://download.fedora.redhat.com/pub/epel/5/$(uname -i)/epel-release-5-4.noarch.rpm
``` bash

Add Cluster Labs repo for Pacemaker
``` bash
wget -O /etc/yum.repos.d/pacemaker.repo http://clusterlabs.org/rpm/epel-5/clusterlabs.repo
```

Install Pacemaker 1.0+ and CoroSync 1.2+
``` bash
yum install -y pacemaker.$(uname -i) corosync.$(uname -i)
```

## Configure CoroSync
Create /etc/corosync/authkey for authentication within cluster communication. This file MUST be copied to each node in the cluster.
If  a message "Invalid digest" appears from the corosync executive, the keys are not consistent between processors.
``` bash
# Generate Key
corosync-keygen

# Define receiving server and copy to it.
node=172.25.3.26
cat /etc/corosync/authkey | ssh root@$node "cat >> /etc/corosync/authkey; chmod 0400 /etc/corosync/authkey; chown hacluster:hacluster /etc/corosync/authkey"
```

Open the example CoroSync configuration:
- Change the `bindnetaddr` to the IP for each server.
- Change `mcastaddr` and `mcastport` to be unique for each cluster.
- Change `secauth` to `on` if encryption is desired or on an insecure network.
The defaults beyond those changes are usually acceptable.
``` bash
cp /etc/corosync/corosync.conf.example /etc/corosync/corosync.conf
vim /etc/corosync/corosync.conf
```

Add in section to tell CoroSync to run as root for resource management. If not clustering resources, leave this out!
```
aisexec {
        # Run as root - this is necessary to be able to manage resources with Pacemaker
        user:        root
        group:       root
}
```
Add in values to start PaceMaker
``` bash
cat <<-END >>/etc/corosync/corosync.conf
service {
    # Load the Pacemaker Cluster Resource Manager
    name: pacemaker
    ver: 0
}
END
```

Once familiarized with the simple structure of it, read the man page. 
It will explain the up-to-date options and what everything means.
``` bash
man corosync.conf.5
```

Copy the configuration to each node in the cluster
``` bash
# Define receiving server and copy to it.
node=172.25.3.26
scp /etc/corosync/corosync.conf root@$node:/etc/corosync/corosync.conf
```

One last thing, as of this writing the RPM packages don't create the necessary folders for the default log file. 
Create them now on all nodes.
``` bash
mkdir /var/log/cluster/

node=172.25.3.26
ssh root@$node "mkdir /var/log/cluster/"
```

Now, start up CoroSync on the first node.
``` bash
service corosync start
```

Check to see CoroSync is running as expected
``` bash
crm_mon
# Or for a single chunk of info:
crm status
```

Since the dark ages of Pacemaker, many changes and advances have been implemented.
Take a look at Cluster from Scratch and page #27 and on. It explains everything very well.
``` bash
crm --help
```

With one node active and functional it is safe to start the second node.
``` bash
node=172.25.3.26
ssh root@$node -- service corosync start
```

## Configure an Active/Passive Cluster [Cluster From Scratch, page 28 and on]
Before we continue we should check if the configuration is correct
``` bash
crm_verify -L
```
As you can see, the tool has found a few errors.

This is due to Pacemaker's use of STONITH. We will configure this later but will disable it for now:
``` bash
crm configure property stonith-enabled=false
crm_verify -L
```
The configuration is now valid.

### Adding a resource: ClusterIP [Cluster From Scratch, page 32]
The first thing we need to do for a cluster is add a resource like an IP address so we can always contact and communicate with the cluster without regardless of where the cluster services are running.
**This must be a NEW address not associated with ANY node.**
``` bash
crm configure primitive ClusterIP ocf:heartbeat:IPaddr2 \
params ip=172.25.3.20 cidr_netmask=21 \
op monitor interval=30s
```

Verify that the IP address has been added to the configuration.
``` bash
crm configure show
crm status
```

We also want to disable autofailback as our machines are generally identical so there is no reason to failback to a specific node.
``` bash
crm configure rsc_defaults resource-stickiness=100
crm configure show
```

Since we normally use a 2 node cluster which is mathematically unable to attain quorum, we need to tell Pacemaker to ignore it.
``` bash
crm configure property no-quorum-policy=ignore
crm configure show
```
Now, if a node goes down the resources will be transitioned appropriately instead of the cluster failing.

### Adding a resource: Apache2 [Cluster From Scratch, page 38]
Install the Apache 2 webserver and `wget` for the cluster to check the status of apache.
``` bash
yum install -y httpd wget
```

Let's create a default index page for Apache to show what server is currently processing requests.
``` bash
# Load the HOSTNAME into our session.
source /etc/sysconfig/network

cat <<-END >/var/www/html/index.html
<html>
    <body>The Apache cluster is on: $HOSTNAME</body>
</html>
END
```

Update Apache configuration to allow /server-status which is what Pacemaker uses to check if Apache is alive.
- Uncomment `ExtendedStatus on`
- Uncomment `</Location /server-status>`
- Update `All from .example.com` to `Allow from 172.25`
``` bash
vim /etc/httpd/conf/httpd.conf
```

Update the configuration so Apache is clustered
``` bash
crm configure primitive WebSite ocf:heartbeat:apache \
params configfile=/etc/httpd/conf/httpd.conf \
op monitor interval=1min

crm configure show
crm status
```

Configure Pacemaker to keep multiple resources together. In this case, Apache can only run on the system with the ClusterIP.
``` bash
crm configure colocation website-with-ip INFINITY: WebSite ClusterIP
crm configure show
crm_mon
```

Configure Pacemaker to start the ClusterIP before WebSite and stop it after WebSite is stopped.
``` bash
crm configure order apache-after-ip mandatory: ClusterIP WebSite
crm configure show
```

We're done! You now have a fully clustered dual or multi node cluster.
Some helpful commands are:
``` bash
# These first 3 are self explanatory:
crm resource start WebSite
crm resource stop WebSite
crm resource restart WebSite

# This allows you to specificy which node in the cluster this resource should be running on.
crm resource move WebSite node2
# After moving, give control back to the cluster
crm resource unmove WebSite

# Set the local node into `standby` which moves all resource off of it allowing for maintenance.
crm node standby
# The same for a remote node
crm node standby node2

# Bring the local or remote node back online after maintenance
crm node online
crm node online node2

# Remove a node from the cluster
crm node delete localhost.localdomain
```

## References:
ClusterLabs [Cluster From Scratch](http://www.clusterlabs.org/doc/Cluster_from_Scratch.pdf)
``` bash
man corosync.conf
man corosync_overview
```

