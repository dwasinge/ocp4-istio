# OpenShift 4 Istio Installation

## Overview

This project is a collection of Ansible inventories for installing the Istio Operator along with it's dependencies.  It also includes installing the Istio sample application BookInfo for testing purposes.

It automates the process of installation defined in the [OpenShift Service Mesh Documentation](https://docs.openshift.com/container-platform/4.1/service_mesh/service_mesh_install/installing-ossm.html).


Currently this has been tested using an OpenShift 4.1 cluster and the tech preview version of the Istio operator which is version 0.12.0.

## How it Works
The layout of the project is like most standard `ansible-playbooks` with a simplified view of the key parts shown below:
```bash
.
├── site.yml
├── requirements.yml
├── inventory
│   ├── host_vars
│   |   └── ...
│   └── hosts
```
 * `site.yml` is a playbook that sets up some variables and drives the `openshift-applier` role.
 * `requirements.yml` is a manifest which contains the Ansible modules needed to run the playbook 
 * `inventory/host_vars/*.yml` is the collection of objects we want to insert into the cluster written according to [the convention defined by the openshift-applier role](https://github.com/redhat-cop/openshift-applier/tree/master/roles/openshift-applier#sourcing-openshift-object-definitions).
 * `inventory/hosts` is where the `targets` are defined for grouping of the various inventories to be run eg `bootstrap` for creating projects and roles bindings
 * `params` is a set of [parameter files](https://docs.openshift.com/container-platform/;latest/dev_guide/templates.html#templates-parameters) to be processed along with their respective OpenShift template.

### Multiple inventories
The Ansible layer is very thin; it simply provides a way to orchestrate the application of [OpenShift templates](https://docs.openshift.com/container-platform/latest/dev_guide/templates.html) across one or more [OpenShift projects](https://docs.openshift.com/container-platform/latest/architecture/core_concepts/projects_and_users.html#projects). All configuration for the applications should be defined by an OpenShift template and the corresponding parameters file or by file.

There are multiple Ansible inventories which divide the type of components to be built and deployed to an OpenShift cluster. These are broken down into multiple sections:
* `jaeger-operator` - Located in `inventory/host_vars/jaeger-operator.yml` contains a collection of objects used to install the Jaeger Operator in OpenShift
* `kiali-operator` - Located in `inventory/host_vars/kiali-operator.yml` contains a collection of objects used to install the Kiali Operator in OpenShift
* `istio-operator` - Located in `inventory/host_vars/istio-operator.yml` contains a collection of objects used to install the Istio Operator in OpenShift
* `control-plane` - Located in `inventory/host_vars/control-plane.yml` creates the namespace and configuration for the Istio Operator to manage in OpenShift
* `sample-app` - Located in `inventory/host_vars/sample-app.yml` creates the objects for the BookInfo application and the VirtualService in OpenShift

## Getting Started

### Prerequisites 

* [Ansible](http://docs.ansible.com/ansible/latest/intro_installation.html) 2.5 or above.
* [OpenShift CLI Tools](https://docs.openshift.com/container-platform/latest/cli_reference/get_started_cli.html)
* Access to the OpenShift cluster (Your user needs permissions to deploy ProjectRequest objects)
* libselinux-python (only needed on Fedora, RHEL, and CentOS)
  - Install by running `yum install libselinux-python`.

### Inventory Usage
It should be noted that non-docker executions will utilize the inventory directory included in this repo by default. If you would like to specify a custom inventory for any of the below tasks, you can do so by adding `-i /path/to/my/inventory` to the command

### Basic Usage

1. Log on to an OpenShift server `oc login -u <user> https://<server>:<port>/`
2. Clone this repository.
3. Install the required [openshift-applier](https://github.com/redhat-cop/openshift-applier) dependency:
```bash
ansible-galaxy install -r requirements.yml --roles-path=roles
```
4. Deploy the Jaeger Operator
```bash
ansible-playbook site.yml -l jaeger-operator
```
5. Deploy the Kiali Operator
```bash
ansible-playbook site.yml -l kiali-operator
```
6. Deploy the Istio Operator
```bash
# Verify that both the Jaeger and Kiali Operators 
# are running before Installing the Istio Operator
ansible-playbook site.yml -l istio-operator
```
7. Deploy the Service Mesh Control Plane
```bash
# Verify that the Istio Operator and the Admission
# Service is running before installing the Control 
# Plane
ansible-playbook site.yml -l control-plane
```
8. Deploy the BookInfo Sample Application
```bash
# Install the sample application once all pods in the
# istio-system namespace are running
ansible-playbook site.yml -l sample-app
```

### Validate the BookInfo is running

Determine the URL for the ingress gateway.
```bash
export GATEWAY_URL=$(oc get route -n istio-system istio-ingressgateway -o jsonpath='{.spec.host}')
```

Call the product page using the gateway from above.
```bash
curl -o  -s -w "%{http_code}\n" http://$GATEWAY_URL/productpage
```

A 200 HTTP Code should be returned from the CURL command.