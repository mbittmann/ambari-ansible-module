## Description
This project contains a custom ansible module for managing an ambari cluster.
It supports creating a cluster, starting all cluster services, stopping all
cluster services, and deleting the cluster. This assumes the existence of
a running ambari server with a set of cluster nodes running the ambari agent.
The custom module is located in `extra_modules/ambari_cluster_state.py`. The `ambari_state.yml` playbook and `roles/ambari_state` role demonstrate how to
use the custom module.

## Defining a blueprint

## Setting the variables

## Running the sample playbook

`ansible-playbook ambari_state.yml`
