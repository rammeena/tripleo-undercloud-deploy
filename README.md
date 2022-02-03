# tripleo-undercloud-deploy

ssh root@undercloud

dnf install git vim nc -y

useradd stack
passwd stack  # specify a password

echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
sudo chmod 0440 /etc/sudoers.d/stack

su - stack

## Hostname set
Ensure that there is a FQDN hostname set and that the $HOSTNAME environment variable matches that value. The easiest way to do this is to set the undercloud_hostname option in undercloud.conf before running the install.

## Download and install the python-tripleo-repos RPM from the appropriate RDO repository
sudo dnf install -y https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-0.1.1-0.20220201181859.87c9879.el8.noarch.rpm
sudo dnf repolist

## Run tripleo-repos to install the appropriate repositories. 
sudo -E tripleo-repos -b wallaby current ceph

## Install the TripleO CLI, which will pull in all other necessary packages as dependencies:
sudo dnf install -y python*-tripleoclient

## Ceph

If you intend to deploy Ceph in the overcloud, or configure the overcloud to use an external Ceph cluster, and are running Pike or newer, then install ceph-ansible on the undercloud:

sudo dnf install -y ceph-ansible

## TLS
If you intend to deploy TLS-everywhere in the overcloud and are deploying Train with python3 or Ussuri+, install the following packages:

sudo yum install -y python3-ipalib python3-ipaclient krb5-devel

## Generate configuration for preparing container images
openstack tripleo container image prepare default   --local-push-destination   --output-env-file /home/stack/tripleo-undercloud-deploy/containers-prepare-parameter.yaml

## Undercloud Install
