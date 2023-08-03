# About

This terraform module is used to provision a server that is part of an opensearch cluster on openstack.

# Assumptions

It assumes that pki authentication will be used (both for node to node communication and for client to server authentication, with optional basic auth authentication supported on the client side) and that most settings not required to bootstrap the cluster (users, roles, tenants, etc) will be generated separately using the opensearch api.

It also assumes that there will two types of nodes in your cluster: Dedicated managers and dedicated workers.

This terraform module has been validated on recent ubuntu images. Your mileage may vary with other distributions.

# Usage

## Input Variables

- **name**: Name to give to the vm. Will be the hostname and name to identity the node in the opensearch cluster as well.
- **image_source**: Source of the image to provision the bastion on. It takes the following keys (only one of the two fields should be used, the other one should be empty):
  - **image_id**: Id of the image to associate with a vm that has local storage
  - **volume_id**: Id of a volume containing the os to associate with the vm
- **flavor_id**: Id of the vm flavor to assign to the instance. See hardware recommendations to make an informed choice: https://etcd.io/docs/v3.4/op-guide/hardware/
- **network_port**: Resource of type **openstack_networking_port_v2** to assign to the vm for network connectivity
- **server_group**: Server group to assign to the node. Should be of type **openstack_compute_servergroup_v2**.
- **keypair_name**: Name of the ssh keypair that will be used to ssh against the vm.
- **chrony**: Optional chrony configuration for when you need a more fine-grained ntp setup on your vm. It is an object with the following fields:
  - **enabled**: If set the false (the default), chrony will not be installed and the vm ntp settings will be left to default.
  - **servers**: List of ntp servers to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#server)
  - **pools**: A list of ntp server pools to sync from with each entry containing two properties, **url** and **options** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#pool)
  - **makestep**: An object containing remedial instructions if the clock of the vm is significantly out of sync at startup. It is an object containing two properties, **threshold** and **limit** (see: https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#makestep)
- **fluentbit**: Optional fluent-bit configuration to securely route logs to a fluend/fluent-bit node using the forward plugin. Alternatively, configuration can be 100% dynamic by specifying the parameters of an etcd store or git repo to fetch the configuration from. It has the following keys:
  - **enabled**: If set the false (the default), fluent-bit will not be installed.
  - **metrics**: Configuration for metrics fluentbit exposes.
    - **enabled**: Whether to enable the metrics or not
    - **port**: Port to expose the metrics on
  - **opensearch_tag**: Tag to assign to logs coming from opensearch
  - **node_exporter_tag** Tag to assign to logs coming from the prometheus node exporter
  - **forward**: Configuration for the forward plugin that will talk to the external fluend/fluent-bit node. It has the following keys:
    - **domain**: Ip or domain name of the remote fluend node.
    - **port**: Port the remote fluend node listens on
    - **hostname**: Unique hostname identifier for the vm
    - **shared_key**: Secret shared key with the remote fluentd node to authentify the client
    - **ca_cert**: CA certificate that signed the remote fluentd node's server certificate (used to authentify it)
- **fluentbit_dynamic_config**: Optional configuration to update fluent-bit configuration dynamically either from an etcd key prefix or a path in a git repo.
  - **enabled**: Boolean flag to indicate whether dynamic configuration is enabled at all. If set to true, configurations will be set dynamically. The default configurations can still be referenced as needed by the dynamic configuration. They are at the following paths:
    - **Global Service Configs**: /etc/fluent-bit-customization/default-config/service.conf
    - **Default Variables**: /etc/fluent-bit-customization/default-config/default-variables.conf
    - **Systemd Inputs**: /etc/fluent-bit-customization/default-config/inputs.conf
    - **Forward Output For All Inputs**: /etc/fluent-bit-customization/default-config/output-all.conf
    - **Forward Output For Default Inputs Only**: /etc/fluent-bit-customization/default-config/output-default-sources.conf
  - **source**: Indicates the source of the dynamic config. Can be either **etcd** or **git**.
  - **etcd**: Parameters to fetch fluent-bit configurations dynamically from an etcd cluster. It has the following keys:
    - **key_prefix**: Etcd key prefix to search for fluent-bit configuration
    - **endpoints**: Endpoints of the etcd cluster. Endpoints should have the format `<ip>:<port>`
    - **ca_certificate**: CA certificate against which the server certificates of the etcd cluster will be verified for authenticity
    - **client**: Client authentication. It takes the following keys:
      - **certificate**: Client tls certificate to authentify with. To be used for certificate authentication.
      - **key**: Client private tls key to authentify with. To be used for certificate authentication.
      - **username**: Client's username. To be used for username/password authentication.
      - **password**: Client's password. To be used for username/password authentication.
  - **git**: Parameters to fetch fluent-bit configurations dynamically from an git repo. It has the following keys:
    - **repo**: Url of the git repository. It should have the ssh format.
    - **ref**: Git reference (usually branch) to checkout in the repository
    - **path**: Path to sync from in the git repository. If the empty string is passed, syncing will happen from the root of the repository.
    - **trusted_gpg_keys**: List of trusted gpp keys to verify the signature of the top commit. If an empty list is passed, the commit signature will not be verified.
    - **auth**: Authentication to the git server. It should have the following keys:
      - **client_ssh_key** Private client ssh key to authentication to the server.
      - **server_ssh_fingerprint**: Public ssh fingerprint of the server that will be used to authentify it.
- **opensearch**: Opensearch configuration. It has the following keys:
  - **cluster_name**: Name of the opensearch cluster. Should be the same for all members of the cluster.
  - **manager**: Whether the ndoe should be a dedicated manager node (otherwise it will be a dedicated worker node).
  - **seed_hosts**: List of manager nodes that the nodes should synchronize to in order to join the cluster. Should be ips or domain names.
  - **bootstrap_security**: Whether the node should bootstrap opensearch security. One and only one node should have this flag set to true when the opensearch cluster is initially created.
  - **initial_cluster**: Whether this node is created as part of the initial cluster that will form opensearch. Nodes that are added to the cluster afterwards should set this to false.
  - **tls**: Parameters to setup tls certificates for networking traffic between cluster members and with clients. It takes the following keys:
    - **ca_certificate**: Certificate of the CA used to sign all other certificates (both for the servers and clients)
    - **server**: Tls credentials for the opensearch nodes
      - **key**: Private tls key
      - **certificate**: Public tls certificate
    - **admin_client**: Tls credentials for an admin client
      - **key**: Private tls key
      - **certificate**: Public tls certificate
  - **auth_dn_fields**: Fields in the certificates that will be used to authentify the admin client and other nodes. Should be the same for all nodes. It is expected to have the following keys:
    - **admin_common_name**: CN value that will identity/authentify the admin user in the admin's client certificate.
    - **node_common_name**: CN value that will identity/authentify this node and other nodes during node-to-node communication.
    - **organization**: Organization value (in the certificat's subject) that will also be used to identify/authentify the admin client and other nodes.
  - **verify_domains**: Whether the domain information in the node certificates should be verified to see if it corresponds to the nodes (that is additional validation on top of the CN validation).
  - **basic_auth_enabled**: Whether basic auth should be enabled as an alternate to certificate authentication as a way to login.
- **install_dependencies**: Whether cloud-init should install external dependencies (should be set to false if you already provide an image with the external dependencies built-in).