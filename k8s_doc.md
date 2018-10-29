## Deploy a Kubernetes cluster manually


### Prerequisite


1. cp kubernetes folder from github.com/reddydodda/mcp_lab

2. cp k8s-vms.yml to infra

3. cp kvm04.yml to infra

4. add classes to infra/config.yml

        - system.reclass.storage.system.kubernetes_control_cluster
        - system.salt.minion.cert.k8s_server
        - cluster.vlab.kubernetes

5. add network hosts confi in infra/config.yml

        infra_kvm_node04:
        classes:
        - cluster.${_param:cluster_name}.infra.kvm04
        domain: ${_param:cluster_domain}
        name: ${_param:infra_kvm_node04_hostname}
        params:
          deploy_address: ${_param:infra_kvm_node04_deploy_address}
          linux_system_codename: xenial
          salt_master_host: ${_param:reclass_config_master}
          single_address: ${_param:infra_kvm_node04_address}
        kubernetes_compute_node01:
        classes:
        - cluster.${_param:cluster_name}.kubernetes.compute
        domain: ${_param:cluster_domain}
        name: ${_param:kubernetes_compute_node01_hostname}
        params:
          linux_system_codename: xenial
        #            salt_master_host: ${_param:infra_config_address}
          single_address: ${_param:kubernetes_compute_node01_address}
          tenant_address: ${_param:kubernetes_compute_node01_tenant_address}
          kubernetes_compute_node02:
          classes:
          - cluster.${_param:cluster_name}.kubernetes.compute
          domain: ${_param:cluster_domain}
          name: ${_param:kubernetes_compute_node02_hostname}
          params:
            linux_system_codename: xenial
        #            salt_master_host: ${_param:infra_config_address}
            single_address: ${_param:kubernetes_compute_node02_address}
            tenant_address: ${_param:kubernetes_compute_node02_tenant_address}
        kubernetes_compute_node03:
          classes:
          - cluster.${_param:cluster_name}.kubernetes.compute
          domain: ${_param:cluster_domain}
          name: ${_param:kubernetes_compute_node03_hostname}
          params:
            linux_system_codename: xenial
        #            salt_master_host: ${_param:infra_config_address}
            single_address: ${_param:kubernetes_compute_node03_address}
            tenant_address: ${_param:kubernetes_compute_node03_tenant_address}
        kubernetes_control_node01:
          name: ${_param:kubernetes_control_node01_hostname}
          params:
            tenant_address: ${_param:kubernetes_control_node01_tenant_address}
        kubernetes_control_node02:
          name: ${_param:kubernetes_control_node02_hostname}
          params:
            tenant_address: ${_param:kubernetes_control_node01_tenant_address}
        kubernetes_control_node03:
          name: ${_param:kubernetes_control_node03_hostname}
          params:
            tenant_address: ${_param:kubernetes_control_node01_tenant_address}



6. add network host in inti.yml
          
        - cluster.vlab.kubernetes

        infra_kvm_node04_address: 10.101.0.244
        infra_kvm_node04_deploy_address: 10.100.0.244
        infra_kvm_node04_hostname: kvm04

        kvm04:
          address: ${_param:infra_kvm_node04_address}
          names:
          - ${_param:infra_kvm_node04_hostname}
          - ${_param:infra_kvm_node04_hostname}.${_param:cluster_domain}


### setup

1. Deploy Salt node

2. Deploy KVM nodes with Drivetrain

3. Update modules and states on all Minions:

        salt '*' saltutil.sync_all

4. Register compute nodes:

        salt-call event.send "reclass/minion/classify" \
        "{\"node_master_ip\": \"<config_host>\", \
        \"node_os\": \"<os_codename>\", \
        \"node_deploy_ip\": \"<node_deploy_network_ip>\", \
        \"node_deploy_iface\": \"<node_deploy_network_iface>\", \
        \"node_control_ip\": \"<node_control_network_ip>\", \
        \"node_control_iface\": \"<node_control_network_iface>\", \
        \"node_tenant_ip\": \"<node_tenant_network_ip>\", \
        \"node_tenant_iface\": \"<node_tenant_network_iface>\", \
        \"node_external_ip\": \"<node_external_network_ip>\", \
        \"node_external_iface\": \"<node_external_network_iface>\", \
        \"node_baremetal_ip\": \"<node_baremetal_network_ip>\", \
        \"node_baremetal_iface\": \"<node_baremetal_network_iface>\", \
        \"node_domain\": \"<node_domain>\", \
        \"node_cluster\": \"<cluster_name>\", \
        \"node_hostname\": \"<node_hostname>\"}"


5. Create and distribute SSL certificates for services using the salt state:

        salt "*" state.sls salt

6. Perform Linux system configuration to synchronize repositories and execute outstanding system maintenance tasks:

       salt -C 'I@docker:host' state.sls linux.system

7. Install Keepalived:

       salt -C 'I@keepalived:cluster' state.sls keepalived -b 1

8. Install HAProxy:

        salt -C 'I@haproxy:proxy' state.sls haproxy
        salt -C 'I@haproxy:proxy' service.status haproxy

9. Install Docker:

       salt -C 'I@docker:host' state.sls docker.host
       salt -C 'I@docker:host' cmd.run "docker ps"

10. Install etcd:

        salt -C 'I@etcd:server' state.sls etcd.server.service
        salt -C 'I@etcd:server' cmd.run "etcdctl cluster-health"

  Install etcd with the SSL support:

      salt -C 'I@etcd:server' state.sls salt.minion.cert,etcd.server.service
      salt -C 'I@etcd:server' cmd.run '. /var/lib/etcd/configenv && etcdctl cluster-health'

11. Install Kubernetes:

        salt -C 'I@kubernetes:master' state.sls kubernetes.master.kube-addons
        salt -C 'I@kubernetes:pool' state.sls kubernetes.pool

12.  For the Calico setup:

  Verify the Calico nodes status:

    salt -C 'I@kubernetes:pool' cmd.run "calicoctl node status"

  Set up NAT for Calico:

    salt -C 'I@kubernetes:master' state.sls etcd.server.setup

13. For the OpenContrail setup, deploy OpenContrail as described in Deploy OpenContrail manually.

14. Run master to check consistency:

        salt -C 'I@kubernetes:master' state.sls kubernetes exclude=kubernetes.master.setup

15. Register addons:

        salt -C 'I@kubernetes:master' --subset 1 state.sls kubernetes.master.setup

16. Restart kubelet:

        salt -C 'I@kubernetes:pool' service.restart kubelet

17. Log in to any Kubernetes ctl node to verify that all nodes have been registered successfully:

        kubectl get nodes
