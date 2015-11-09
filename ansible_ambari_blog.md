## Custom Ansible Module for Managing Ambari Clusters
At B23, we believe in the use of automation for provisioning and configuring cloud resources to manage complex data pipelines, whether streaming or bulk processing. Data pipelines have evolved beyond the pattern of writing blocks of data to HDFS and running bulk operations. MapReduce is a freight truck, but you might need 5 Jeeps, 2 Jaguars, and a helicopter.

There now exist many tools for managing asynchronous, distributed data pipelines, with each tool designed for particular access patterns. These include Kafka, Storm, Spark, MapReduce, Hive, Elasticsearch, a handful of NoSQL databases, and yes the old fashioned relational database. While we are left with the paradox of choice, these options enable us to build precision data products tuned for a particular business problem. Systems often require multiple compute paradigms and access patterns. We use automation and cloud to reduce the complexity of such systems while at the same time enforcing strict policies for security and compliance.

At B23, we've come to know Ansible very well for instantiating and configuring distributed data products. We often use Apache Ambari for installing and configuring software stacks in the Hadoop ecosystem. It doesn't manage every tool, but it really simplifies the configuration of the Hadoop ecosystem. Ambari has a very flexible RESTful API and a powerful UI. However, we found ourselves generating a lot of overlapping [Ambari Blueprints](https://cwiki.apache.org/confluence/display/AMBARI/Blueprints). We also found ourselves repeating a lot of ansible `uri` and `wait_for` calls to create, stop, start, and delete clusters in Ambari. Lastly, we were using a lot of Jinja2 to inject hostnames pulled midstream from Ansible's [dynamic inventory](http://docs.ansible.com/ansible/intro_dynamic_inventory.html). To overcome this, we developed a [custom ansible module module](http://docs.ansible.com/ansible/developing_modules.html) for managing clusters, and B23 is pleased to make it available as open source:

https://github.com/mbittmann/ambari-ansible-module

I think the module has several benefits. First is the simplification of the blueprint. Right now, you can define an Ambari cluster in two JSON files: the `blueprint` and the `host_map`. The `blueprint` defines a generic cluster by mapping a set of cluster components (e.g., Namenode, Nimbus server) to groups (e.g., master, worker). The `host_map` then maps a set of hosts to those groups. While JSON is the language of REST, YAML is the language of ansible, and so our custom module allows you to define a cluster with the compact, readable form of YAML. Here is an example blueprint defined entirely as Ansible variables:

```
master_hosts: [host1]
slave_hosts: [host2, host3, host4]

blueprint:
  stack_name: HDP
  stack_version: 2.2
  groups:
    - name : master
      cardinality: 1
      configuration: []  # configuration not yet implemented
      components:
        - NAMENODE
        - SECONDARY_NAMENODE
        - RESOURCEMANAGER
        - HISTORYSERVER
        - GANGLIA_SERVER
        - ZOOKEEPER_SERVER
      hosts: "{{ master_hosts}}"
    - name: slaves
      cardinality: 1+
      configuration: []  # configuration not yet implemented
      components:
        - APP_TIMELINE_SERVER
        - DATANODE
        - HDFS_CLIENT
        - NODEMANAGER
        - YARN_CLIENT
        - MAPREDUCE2_CLIENT
        - ZOOKEEPER_CLIENT
      hosts: "{{ slave_hosts }}"
```

Because we are defining our cluster with Ansible variables, we can further simplify and reuse cluster definitions by defining the overall service-component mappings as variables:

```
hadoop_master: [NAMENODE, SECONDARY_NAMENODE, RESOURCEMANAGER, HISTORYSERVER]
hadoop_slave: [APP_TIMELINE_SERVER, DATANODE, HDFS_CLIENT, NODEMANAGER, YARN_CLIENT, MAPREDUCE2_CLIENT]
zookeeper_master: [ZOOKEEPER_SERVER]
zookeeper_slave: [ZOOKEEPER_CLIENT]
ganglia_master: [GANGLIA_SERVER]

master_components: "{{ hadoop_master | union(zookeeper_master) | union(ganglia_master) }}"
slave_components: "{{ hadoop_slave | union(zookeeper_slave) }}"
```

This allows us to express a cluster in the following concise and completely dynamic template:
```
blueprint:
  stack_name: HDP
  stack_version: 2.2
  groups:
    - name : master
      cardinality: 1
      configuration: []  # configuration not yet implemented
      components: "{{ master_components }}"
      hosts: "{{ master_hosts }}"
    - name: slaves
      cardinality: 1+
      configuration: []  # configuration not yet implemented
      components: "{{ slave_components }}"
      hosts: "{{ slave_hosts }}"
```

Now consider the use of Ansible dynamic inventory. We can use EC2 tags to find all hosts with an `Ambari_Group` tag matching `slave` or `master` and populate the `slave_hosts` and `master_hosts` variables without needing to mess with changing hostnames or IP addresses.

Another advantage is that we included a module parameter to wait for a cluster request to complete. This moves the logic of waiting for Ambari to install, configure, and start all service components out of Ansible and into the python module. This is valuable in case you need all Spark services to be running before starting some Spark job in a handler.

The module does not yet support injecting service or component configuration into Ambari, which is a pretty killer feature of blueprints. We have plans to add that in the future.

The introduction of this module into our workflows has significantly reduced the amount of playbook logic and template files in our codebase. It also enables us to more quickly integrate systems that incorporate multiple tools in the ever growing big data ecosystem.  
