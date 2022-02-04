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
```
openstack undercloud install
```
# Prepare Overcloud CentOS 8 steam images
```
export STABLE_RELEASE="wallaby"
export OS_YAML="/usr/share/openstack-tripleo-common/image-yaml/overcloud-images-centos8.yaml"
export DIB_YUM_REPO_CONF="/etc/yum.repos.d/delorean* /etc/yum.repos.d/tripleo-centos-*"
export DIB_RELEASE=8-stream
openstack overcloud image build --config-file /usr/share/openstack-tripleo-common/image-yaml/overcloud-images-python3.yaml --config-file $OS_YAML
openstack overcloud image upload
```

## Patch Ironic Conductor for ovirt sdk

Add below section in containers-prepare-parameter.yaml file
  - push_destination: true
    includes:
    - ironic-conductor
    modify_role: tripleo-modify-image
    modify_append_tag: "-ovirt"
    modify_vars:
      tasks_from: rpm_install.yml
      rpms_path: /home/stack/tripleo-undercloud-deploy/ironic_hotfix
      
Ensure the below packages are available on rpms_path location in above code snippet

(undercloud) [stack@undercloud ~]$ ls -l /home/stack/tripleo-undercloud-deploy/ironic_hotfix
total 800
-rw-rw-r--. 1 stack stack 584132 Feb  3 06:31 python3-ovirt-engine-sdk4-4.4.15-1.el8.x86_64.rpm
-rw-rw-r--. 1 stack stack 232868 Feb  3 06:31 python3-pycurl-7.43.0.2-4.el8.x86_64.rpm

Then run below command:

openstack tripleo container image prepare -e ~/tripleo-undercloud-deploy/containers-prepare-parameter.yaml --deub

make sure that ~/tripleo-undercloud-deploy/containers-prepare-parameter.yaml contains above snippet which patches ironic conductor for ovirt sdk.

# patch ironic-conductor for another error in ovirt sdk
Error when registering a new baremetal node: TypeError: startswith first arg must be bytes or a tuple of bytes, not str

on undercloud node search for ovirt.py in /usr/lib/python3.6/site-packages/ironic_staging_drivers/ovirt/ directory.

This file needs to be patched for ironic-conductor container:
-------------------------------------------------------------

find / -name 'ironic_staging_drivers' -type d|grep  merged

it will be found in something like below directotry:

  vim /var/lib/containers/storage/overlay/4a813aa1947e62b2487a3b164113fd0c89f8fc878fccfcced756a2249c939979/merged/usr/lib/python3.6/site-packages/ironic_staging_drivers/ovirt/ovirt.py

replace the content of this file with below patced file:
  https://review.opendev.org/c/x/ironic-staging-drivers/+/784879/6/ironic_staging_drivers/ovirt/ovirt.py#147

file location: 
https://review.opendev.org/plugins/gitiles/x/ironic-staging-drivers/+/refs/changes/79/784879/6/ironic_staging_drivers/ovirt/ovirt.py

However this file is available in this repo as well, just copy this in above container file location:

cp -v ovirt.py /var/lib/containers/storage/overlay/4a813aa1947e62b2487a3b164113fd0c89f8fc878fccfcced756a2249c939979/merged/usr/lib/python3.6/site-packages/ironic_staging_drivers/ovirt/ovirt.py

Now restart ironic conductor podman container.

Now baremetal node enrollement should work fine.
