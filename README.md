## Description
This project contains a custom ansible module for managing an ambari cluster.
It supports creating a cluster, starting all cluster services, stopping all
cluster services, and deleting the cluster. This assumes the existence of
a running ambari server with a set of cluster nodes running the ambari agent.
The custom module is located in `extra_modules/ambari_cluster_state.py`. The `ambari_state.yml` playbook and `roles/ambari_state` role demonstrate how to
use the custom module.

## Motivation
The power of dynamic inventory variables in ansible make it an excellent tool
for managing cloud infrastructure. Ambari does much of the hard work in
installing and configuring the hadoop ecosystem. This custom module helps us
automate and simplify the management of dynamically provisioned hadoop clusters. 

## Defining a blueprint
The standard for ambari is to define a blueprint as a JSON file. This file maps
ambari services (e.g. namenode, datanode) to user defined host groups. Then,
you can map hosts to those same host groups as a separate JSON structure known
as a host map. In the spirit of ansible, I combine these two structures into a
more concise yaml file. See `roles/ambari_state/files/blueprint.yml` for an
example. If useful, it would pretty simple to support the use of existing
blueprint/host map files as well.

## Running the sample playbook
Specify the appropriate vars in `ambari_state.yml` for your ambari server, and
specify the appropriate services, groups, and hosts in `roles/ambari_state/files/blueprint.yml`

`ansible-playbook ambari_state.yml`
