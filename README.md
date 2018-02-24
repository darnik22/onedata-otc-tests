# Terraform/ansible/helm files for preparing testing environment for Onedata on OTC

Repo structure:
* kube-centos - terraform and ansible files for creating infrastructure and deploying kubernetes cluster and VPN server. 
* ceph4kube-centos - terraform and ansible files for creating infrastructure and deploying ceph cluster in the same VPC as the kubernetes cluster
* charts - consists of helm charts for deploying onedata
* scale-3p-land.yaml - helm landscape for deploying onedata

## Configuring k8s cluster

In order to build your cluster you need to:
* run ssh-agent and add your key. It will further be used to login into the created VMs.
* provide your openstack credentials by editting parameter.tvars. The username should be the same as shown in the OTC console. You can not use the email or mobile number, which can also be used to login to the OTC web console. 
* eventually change values in varaibles.tf according to the comments in this file.

The testing environment uses public DNS. It is required to have a public domain under control. In order to integrate kube-dns with your public domain you need to:
  * have a registered Internet domain which uses (delegates to) the following nameservers:
  * ns1.open-telekom-cloud.com.
  * ns2.open-telekom-cloud.com.

Edit kube-centos/variables.tf and set at least the following vars:
* dnszone - your registered Internet domain. The publicly resolvable cluster domain will be kube.${dnszone}.
* project - project name. It is used to prefix VM names. It should be unique among OTC as it is used to create names of VMs.
* public_key_file - the path to your ssh public key.
You can also set the variables specifying the networks to be used (in case of conficts with the existing ones):
* kube_service_addresses
* kube_pods_subnet
* vpn_network
* vpc_subnet


The variables can also be provided interactively or set as command line args. For example:
```
terraform apply -var project=example_project -var email=joe@example.com ....
```

## Configuring the Ceph cluster

Edit ceph4kube-centos/variables.tf and set the variables for the Ceph cluster. Note that the values of related variables (project, dnszone) should be the same as for the k8s cluster. 

## Running
Build your k8s cluster issuing:
```
cd kube-centos
terraform init
terraform apply -var-file ../parameter.tvars
```

After a successful build go to the ceph4kube-centos directory and run terraform using the same arguments:

```
cd ../ceph4kube-centos
terraform init
terraform apply -var-file ../parameter.tvars
```

## Accessing your k8s cluster
After a successful build the public and private IPs of the k8s cluster master node are displayed. In order to have access to the private addresses you need to connect to the VPN server, which has been prepared. If the terraform has been run from a linux machine then on that machine a VPN client has been run already by the scripts. You can also use the "scp ubuntu@..." commands displayed after successful terraform execution to manually start a VPN connection and kube proxy on another machine, e.g., your laptop or desktop. An example commands looks like:
```
scp ubuntu@80.158.20.236:laptop.sh .; ./laptop.sh
```

## Configuring the landscape

Create a SFS (Scalable File Service) share on OTC for the VPC which has been created by terraform. It is called {{project}}-router. Replace {{project}} with the value of project variable you specified earlier. Note the SFS share id and place it in charts/volume-sfs/values.yaml. Eventually modify the size, which defaults to 3T. Do not create volumes with less than 3000GB sizes.

Edit the file scale-3p-land.yaml and change tld to your domain.

## Grafana

Grafana URL is http://grafana.mon.svc.kube.{{dnszone}}:80/. An active VPN connection is needed to view it. A basic dashboard "oc-trans-rate" has been uploaded. It shows oneclients aggregated read and write rates for ceph. 

## Deploying onedata

Login to the kubernetes master node:
```
ssh {{project}}-kube-ctlr.{{dnszone}} -l linux
```
Run the following command to deploy onedata:
```
helm install -f scale-3p-land.yaml charts/cross-support-job-3p -n st
```
Observe the progress of the deployment and wait until all pods reach the Running status:
```
watch kubectl get pod
```
Use a browser to login to onezone and check the deployment was successful. The URL is https://st-onezone.default.svc.kube.{{dnszone}}/ and credentials are admin:password.

## Preparing and running a simple job

A kubernetes job definition file "wr-test-job.yaml" has been uploaded to the master node. Copy an access token from onezone and place it in this file. Run the job:
```
kubectl create -f wr-test-job.yaml
```

## Example command flow 
```
eval `ssh-agent`
ssh-key
git clone https://github.com/darnik22/onedata-otc-tests.git
cd onedata-otc-tests
vi parameter.tvars
cd kube-centos
vi variables.tf
terraform init
terraform apply -var-file ../parameter.tvars -var project=myproject -var dnszone=mydnszone
cd ../ceph4kube-centos
vi variables.tf
terraform init
terraform apply -var-file ../parameter.tvars -var project=myproject -var dnszone=mydnszone

ssh -A ubuntu@THE_IP_OF_CTLR_NODE
kubectl cluster-info
kubectl get pod

```

SFS: Create SFS for the VPC after creating infrastructure and before helming a lanscape 