# PCF+NSX POC Using VMware Fusion and Photon OS
## Step-by-step instructions

## Requirements
- Mac laptop with VMware Fusion 8.0 or higher
- 4 GB of RAM available
- 20 GB of disk space
- Internet connection

## Process
### Environment preparation
1. Download the latest version of VMware Photon OS from [here](https://github.com/vmware/photon/wiki/Downloading-Photon-OS). Select `OVA with virtual hardware v11`.
2. Import the OVA as a virtual machine following [these instructions](https://github.com/vmware/photon/wiki/Running-Project-Photon-on-Fusion).
3. Assign 20 GB of dynamic disk, 2 GB of RAM and 2 vCPUs.
4. Boot the VM and login as `root/changeme`. You will be prompted for a new password upon the first login.
5. Find out your IP address with `ifconfig eth0`.
6. From your Mac terminal (or [iTerm](https://iterm2.com/), which is a better app altogether), ssh into your VM as root. This makes the process more practical to copy-and-paste larger blocks in this guide.
7. As root, enable Docker and make it persistent: `systemctl start docker && systemctl enable docker`
8. Install some required dependencies in the VM itself: `tdnf install git wget gawk`. Say yes to any question.
9. Download and install Docker Compose:

```
curl -L https://github.com/docker/compose/releases/download/1.12.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
 
At this point, the base environment is ready to install and run Concourse pipelines.
 
### Install and setup Concourse
Execute the following commands in sequence:

```
mkdir concourse && cd concourse
mkdir -p keys/web keys/worker
ssh-keygen -t rsa -f ./keys/web/tsa_host_key -N ''
ssh-keygen -t rsa -f ./keys/web/session_signing_key -N ''
ssh-keygen -t rsa -f ./keys/worker/worker_key -N ''
cp ./keys/worker/worker_key.pub ./keys/web/authorized_worker_keys
cp ./keys/web/tsa_host_key.pub ./keys/worker
export CONCOURSE_EXTERNAL_URL=http://$(ifconfig eth0 | awk '/inet addr/ {gsub("addr:", "", $2); print $2}'):8080
```

Next, create the file `docker-compose.yml` in the same concourse directory with the following content:

```
concourse-db:
  image: postgres:9.5
  environment:
    POSTGRES_DB: concourse
    POSTGRES_USER: concourse
    POSTGRES_PASSWORD: changeme
    PGDATA: /database

concourse-web:
  image: concourse/concourse
  links: [concourse-db]
  command: web
  ports: ["8080:8080"]
  volumes: ["./keys/web:/concourse-keys"]
  environment:
    CONCOURSE_BASIC_AUTH_USERNAME: pcf
    CONCOURSE_BASIC_AUTH_PASSWORD: pcf
    CONCOURSE_EXTERNAL_URL: "${CONCOURSE_EXTERNAL_URL}"
    CONCOURSE_POSTGRES_DATA_SOURCE: |-
      postgres://concourse:changeme@concourse-db:5432/concourse?sslmode=disable

concourse-worker:
  image: concourse/concourse
  privileged: true
  links: [concourse-web]
  command: worker
  volumes: ["./keys/worker:/concourse-keys"]
  environment:
    CONCOURSE_TSA_HOST: concourse-web
```

In this example, the default login for Concourse will be `pcf` with password `pcf`. Change accordingly. If the login and password for Postgress are to be changed (not needed for a POC), make sure they are changed in both sections.

You can now run the Concourse web server for first time with: `docker-compose up`. The latest Docker images should be downloaded and ran automatically. Once you can determine Concourse is running, you can verify that by logging in via the web UI at `http://<ip-of-the-VM>:8080` with the `pcf/pcf` credentials from above in the default team called `main`. You should see no pipelines configured.

To run this command again after a reboot, make sure you define your `CONCOURSE_EXTERNAL_URL` variable again. A more permanent solution would be to create a service via `systemd` configuration, but that's beyond the goal of this guide.

### Download the `fly` CLI

The latest binaries for the `fly` CLI to control Concourse can be found [here](http://concourse.ci/downloads.html). Be careful to download the *Fly Binaries*, not the *Concourse Binaries*.

Copy the URL of the Linux version (in this example the latest version available is v2.7.3) and download it as follows:

```
wget https://github.com/concourse/concourse/releases/download/v2.7.3/fly_linux_amd64
mv fly_linux_amd64 /usr/local/bin/fly
chmod +x /usr/local/bin/fly
```

### Download and configure the Pipeline
Clone the latest pipeline and create an empty parameters directory as follows:

```
git clone https://github.com/cf-platform-eng/nsx-ci-pipeline.git
mkdir nsx-ci-pipeline/params
```

Edit the new file `nsx-ci-pipeline/params/env1-params.yml`.
Copy and paste the entire content of the next block into the file:

```
#########################
## ERT & Opsman config ##
#########################
---
# Core Concourse Resource Params
pivnet_token: <YOUR PIVNET TOKEN> #REQUIRED
github_user: <YOUR GITHIB USERID> #REQUIRED
github_token: <YOUR GITHIB TOKEN> #REQUIRED

## vCenter Params
vcenter_host: <YOUR VCENTER URL|IP> #REQUIRED
vcenter_usr: administrator@vsphere.local #REQUIRED
vcenter_pwd: <YOUR VCENTER ADMIN PASSWD> #REQUIRED
vcenter_data_center: Datacenter #REQUIRED

## NSX Integration Params
nsx_edge_gen_nsx_manager_address: <YOUR NSX MANAGER URL|IP> #REQUIRED
nsx_edge_gen_nsx_manager_admin_user: admin #REQUIRED
nsx_edge_gen_nsx_manager_admin_passwd: <YOUR NSX MANAGER PASSWORD> #REQUIRED
nsx_edge_gen_nsx_manager_transport_zone: <YOUR NSX TRANSPORT ZONE> #REQUIRED
nsx_edge_gen_egde_datastore: <YOUR DATASTORE FOR NSX EDGES> #REQUIRED example: vsanDatastore
nsx_edge_gen_egde_cluster: <YOUR CLUSTER FOR NSX EDGES> #REQUIRED example: Cluster1
nsx_edge_gen_name: nsx-pipeline-sample #string name for NSX objects
esg_size: compact
esg_cli_username_1: admin 
esg_cli_password_1: P1v0t4l!P1v0t4l!
esg_certs_name_1: nsx-gen-created
esg_default_uplink_pg_1: "<YOUR NSX-EDGE-UPLINK PORT GROUP>" #REQUIRED "VM Network"
esg_default_uplink_ip_1: <YOUR NSX-EDGE-PRIMARY-VIP> #REQUIRED example: 10.172.16.100
esg_opsmgr_uplink_ip_1: <YOUR OPSMAN-VIP> #REQUIRED example: 10.172.16.101
esg_go_router_uplink_ip_1: <YOUR ERT-VIP> #REQUIRED example: 10.172.16.102
esg_diego_brain_uplink_ip_1: <YOUR SSH-PROXY-VIP> #REQUIRED example: 10.172.16.103
esg_tcp_router_uplink_ip_1: <YOUR TCP-ROUTER-VIP> #REQUIRED example: 10.172.16.104
esg_gateway_1: <YOUR ROUTED-UPLINK-NETWORK GATEWAY> #REQUIRED example: 10.172.16.1


#### Opsman configuration
## Ops Manager installation meta data
om_data_store: vsanDatastore #REQUIRED
om_host: <YOUR FQDN DNS FOR OPSMAN VIP> #REQUIRED example: opsman.domain.local
om_usr: admin
om_pwd: P1v0t4l!
om_ssh_pwd: P1v0t4l!
om_decryption_pwd: P1v0t4l!
om_ntp_servers: <YOUR ENVIRONMENTS NTP> #REQUIRED example: 10.193.99.2
om_dns_servers: <YOUR ENVIRONMENTS DNS> #REQUIRED example: 10.193.99.2
om_gateway: 192.168.10.1
om_netmask: 255.255.255.0
om_ip: 192.168.10.5

om_vm_network: nsxgen
om_vm_name: opsman-nsx-pipeline
om_resource_pool: <YOUR TARGET RESPOOL FOR OPSMAN> #REQUIRED example: respool-opsman

disk_type: thin
om_vm_power_state: true

storage_names: <YOUR TARGET DATASTORE(S) FOR PCF> #REQUIRED example: vsanDatastore,vsanDatastore2,vsanDatastore3

## AZ configuration for Ops Director, as configured requires 3 vSphere cluster|respools
az_1_name: az1
az_2_name: az2
az_3_name: az3
az_singleton: az1
az_ert_singleton: az1
azs_ert: az1,az2,az3

az_1_cluster_name: <YOUR AZ1 CLUSTER> #REQUIRED example: Cluster1
az_2_cluster_name: <YOUR AZ2 CLUSTER> #REQUIRED example: Cluster2
az_3_cluster_name: <YOUR AZ3 CLUSTER> #REQUIRED example: Cluster3

az_1_rp_name: <YOUR AZ1 RESPOOL> #REQUIRED example: cc-pipeline-rp1
az_2_rp_name: <YOUR AZ1 RESPOOL> #REQUIRED example: cc-pipeline-rp2
az_3_rp_name: <YOUR AZ1 RESPOOL> #REQUIRED example: cc-pipeline-rp3

ntp_servers: <YOUR ENVIRONMENTS NTP> #REQUIRED example: 10.193.99.2
ops_dir_hostname:

## Network configuration for Ops Director
infra_network_name: "INFRASTRUCTURE"
infra_vsphere_network: nsxgen
infra_nw_cidr: 192.168.10.0/26
infra_excluded_range: 192.168.10.1-192.168.10.9,192.168.10.60-192.168.10.61
infra_nw_dns: <YOUR INFRA NET DNS> #REQUIRED
infra_nw_gateway: 192.168.10.1
infra_nw_az: az1,az2,az3

deployment_network_name: "ERT"
deployment_vsphere_network: nsxgen
deployment_nw_cidr: 192.168.20.0/22
deployment_excluded_range: 192.168.20.1-192.168.20.9,192.168.23.250-192.168.23.253
deployment_nw_dns: <YOUR ERT NET DNS> #REQUIRED
deployment_nw_gateway: 192.168.20.1
deployment_nw_az: az1,az2,az3

services_network_name: "PCF-TILES"
services_vsphere_network: nsxgen
services_nw_cidr: 192.168.24.0/22
services_excluded_range: 192.168.24.1-192.168.24.9,192.168.27.250-192.168.27.253
services_nw_dns: <YOUR PCF TILES NET DNS> #REQUIRED
services_nw_gateway: 192.168.24.1
services_nw_az: az1,az2,az3

dynamic_services_network_name: "PCF-DYNAMIC-SERVICES"
dynamic_services_vsphere_network: nsxgen
dynamic_services_nw_cidr: 192.168.28.0/22
dynamic_services_excluded_range: 192.168.28.1-192.168.28.9,192.168.31.250-192.168.31.253
dynamic_services_nw_dns: <YOUR PCF DYN-SERVCIES NET DNS> #REQUIRED
dynamic_services_nw_gateway: 192.168.28.1
dynamic_services_nw_az: az1,az2,az3

loggregator_endpoint_port: 443

#### ERT configuration
## ERT Syslog endpoint configuration goes here
syslog_host:
syslog_port:
syslog_protocol:

network_point_of_entry: external_non_ssl

## ERT Wildcard domain certs go here
ssl_cert:
ssl_private_key:

disable_http_proxy: true

## ERT TCP routing and routing services
tcp_routing: enable
tcp_routing_ports: 5000
route_services: enable
ignore_ssl_cert_verification: true

## ERT SMTP configuration goes here
smtp_from:
smtp_address:
smtp_port:
smtp_user:
smtp_pwd:
smtp_auth_mechanism:

## ERT Auth Config method
uaa_method: internal

## ERT LDAP Configuration goes here
ldap_url:
ldap_user:
ldap_pwd:
search_base:
search_filter:
group_search_base:
group_search_filter:
mail_attribute_name:
first_name_attribute:
last_name_attribute:

## ERT Deployment domain names
system_domain: <YOUR WILDCARD DNS MAPPED TO ERT VIP FOR SYSTEM URL> #REQUIRED example: sys.domain.local
apps_domain: <YOUR WILDCARD DNS MAPPED TO ERT VIP FOR DEFAULT APPS URL> #REQUIRED example: apps.domain.local

skip_cert_verify: true

## ERT Static IP's for the following jobs
ha_proxy_ips:
router_static_ips: 192.168.20.10,192.168.20.11,192.168.20.12,192.168.20.13
tcp_router_static_ips: 192.168.20.30,192.168.20.31,192.168.20.32,192.168.20.33
ssh_static_ips: 192.168.20.20,192.168.20.21,192.168.20.22
ert_mysql_static_ips: 192.168.20.41,192.168.20.42,192.168.20.43

## ERT Target email address to receive mysql monitor notifications
mysql_monitor_email: <SMTP FOR MYSQL ALERTS> #REQUIRED example: someone@example.com

## ERT Default resource configuration
consul_server_instances: 1
nats_instances: 1
etcd_tls_server_instances: 1
nfs_server_instances: 1
mysql_proxy_instances: 1
mysql_instances: 1
backup_prepare_instances: 0
ccdb_instances: 0
uaadb_instances: 0
uaa_instances: 1
cloud_controller_instances: 1
ha_proxy_instances: 0
router_instances: 1
mysql_monitor_instances: 1
clock_global_instances: 1
cloud_controller_worker_instances: 1
diego_database_instances: 1
diego_brain_instances: 1
diego_cell_instances: 3
doppler_instances: 1
loggregator_traffic_controller_instances: 1
tcp_router_instances: 1

##################
## MYSQL config ##
##################
tile_az_mysql_singleton: az1
tile_azs_mysql: az1,az2,az3
tile_mysql_proxy_ips: 192.168.24.10,192.168.24.11,192.168.24.12
tile_mysql_proxy_vip: #Hardcoded to 192.168.27.250 in nsxgen leave blank
tile_mysql_monitor_email: <SMTP FOR MYSQL ALERTS> #REQUIRED example: someone@example.com

###################
## Rabbit config ##
###################
tile_az_rabbit_singleton: az1
tile_azs_rabbit: az1,az2,az3
tile_rabbit_proxy_ips: 192.168.24.30,192.168.24.31,192.168.24.32
tile_rabbit_proxy_vip: #Hardcoded to 192.168.27.251 in nsxgen leave blank
tile_rabbit_admin_user: rabbitadmin
tile_rabbit_admin_passwd: rabbitadmin

###################
## SCS config ##
###################
tile_az_scs_singleton: az1
tile_azs_scs: az1,az2,az3
```

It's very important that all the proper variables are correctly configured representing the IPs and names of your specific vSphere and NSX environments, which must also be reachable.

Pivnet refers to the credentials to download releases from `network.pivotal.io`. You need to create an account on the service and obtain your token from the bottom of your [profile page](https://network.pivotal.io/users/dashboard/edit-profile).

Only once you are completely confident you are done with this configuration, move to the next step.

### Running the pipeline

```
cd
fly --target photon_vm login --concourse-url http://<ip>:8080
fly -t photon_vm set-pipeline -n -p pcf -c nsx-ci-pipeline/pipelines/new-setup-with-nsx-edge-gen/pipeline.yml -l nsx-ci-pipeline/params/env1-params.yml
```

This will load the pipeline into Concourse. It will appear in `paused` state initially. At this point you should be able to see the pipeline in the Web UI. To run it, you need to "unpause" it with:

```
fly -t photon_vm unpause-pipeline -p pcf
```

After a few seconds, you should see the pipeline running in the Web UI. If there is any error, at any point we can inspect any task and determine what's wrong, make the proper changes, and run the pipeline again.