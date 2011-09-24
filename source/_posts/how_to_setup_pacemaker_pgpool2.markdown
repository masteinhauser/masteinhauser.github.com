## How To: Install Pacemaker + PgPool-II 3.1 on CentOS 5.7
This is the second post on the clustering system I have been a part of designing and implementing. In this post I will be walking through how to build a highly available instance of pgPool-II 3.1. If you don't know what pgPool-II is, it is a load balancing, connection pooling and database replication system for PostgreSQL. We use it for when Bad Things Happen and we don't want customers to notice. Currently, this guide won't walk you through installing pgPool-II, but it will walk you through making a current install clustered and highly available.
<!--more-->
Refer back to my previous guide of [How To Install CoroSync with Pacemaker and Apache 2]() for the initial steps to create the cluster.
Refer to [Installing pgPool-II on CentOS](http://pgpool.projects.postgresql.org/pgpool-II/doc/tutorial-en.html#install) for steps on how to install and configure pgPool-II

Download and save this into `/usr/lib/ocf/resource.d/heartbeat/pgpool2`
{% include_code Save to /usr/lib/ocf/resource.d/heartbeat/ pgpool2 lang:bash %}

Update the configuration so pgPool-II is clustered
**Change the parameters below to match your environment!**
``` bash
crm configure primitive pgPool ocf:heartbeat:pgpool2 \
params pcp_admin_username=postgres \
params pcp_admin_password=postgres \
params pcp_admin_port=9898 \
params pcp_admin_host=localhost \
params pgpool_bin=/usr/sbin/pgpool \
params pcp_attach_node_bin=/usr/bin/pcp_attach_node \
params pcp_detach_node_bin=/usr/bin/pcp_detach_node \
params pcp_node_count_bin=/usr/bin/pcp_node_count \
params pcp_node_info=/usr/bin/pcp_node_info \
params stop_mode=f \
params auto_reconnect=t \
op monitor interval=1min

crm configure show
crm status
```

Configure Pacemaker to keep multiple resources together. In this case, pgPool-II can only run on the system with the ClusterIP.
``` bash
crm configure colocation pgpool-with-ip INFINITY: pgPool ClusterIP
crm configure show
crm_mon
```

Configure Pacemaker to start the ClusterIP before pgPool and stop it after pgPool is stopped.
``` bash
crm configure order pgPool-after-ip mandatory: ClusterIP pgPool
crm configure show
```

## References:
[Installing pgPool-II on CentOS](http://pgpool.projects.postgresql.org/pgpool-II/doc/tutorial-en.html#install)
ClusterLabs [Cluster From Scratch](http://www.clusterlabs.org/doc/Cluster_from_Scratch.pdf)

