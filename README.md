# Monitoring OpenShift PODS with Ansible and Zabbix Sender 
This smart-start describes the creation  an OpenShift project to monitoring PODS using Zabbix Sender and Ansible

## Prerequisites
* OpenShift or generic kubernetes cluster
* OpenShift client (oc) or kubectl
* OpenJDK 11 or Graalvm-11
* Zabbix Server

## Summary

* [Creating OpenShift Projects](#creating-openshift-projects)
  * [Monitoring API Project](#monitoring-api-project])
    * [Build ansible-agent4ocp](#build-ansible-agent4ocp)
    * [Pushing image](#pushing-image)
  * [APIs project](#apis-project)
    * [Deploying custumer-api](#deploying-custumer-api)
    * [Deploying inventory-api](#deploying-inventory-api)


## Creating OpenShift Projects
In this session we'll create an OpenShift projects to deploy apis example and CronJobs to get container metrics and send to Zabbix Server.

### Monitoring API Project
In this project we'll deploy ansible-agent4ocp to monitoring APIs PODs.

```bash
$ oc new-project apis-monitoring
```
```console
Now using project "apis-monitoring" on server "https://api.shared-na46.openshift.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.
```
#### Build ansible-agent4ocp
This is the containerfile for ansible-agent4ocp
```dockerfile
FROM quay.io/centos/centos:stream8 #base image

USER root

ENV HOME=/opt/scripts #workdir folder

WORKDIR ${HOME}  

RUN yum install epel-release -y && \ #CentoOS Stream extras repositories
    yum update -y && \ #update SO
    yum install ansible.noarch -y && \ #intall ansible
    yum clean all && \
    curl https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz --output /tmp/oc.tar.gz && \ #download and install OpenShift Client
    tar xvzf /tmp/oc.tar.gz && \
    cp oc /usr/local/bin && \
    rm oc kubectl && \
    rm /tmp/oc.tar.gz && \
    mkdir -pv ${HOME} && \  #Granting permissions to folders and files
    mkdir -pv ${HOME}/.ansible/tmp && \
    mkdir -pv ${HOME}/.kube/ && \
    mkdir -pv ${HOME}/playbooks && \
    chown -R 1001:root ${HOME} &&  \
    chgrp -R 0 ${HOME} && \
    chmod -R g+rw ${HOME} 

VOLUME ${HOME}/playbooks #folder to save playbooks

USER 1001

ADD example.yml ${HOME}/example.yml #ansible example file
```



#### Pushing image

### APIs project

#### Deploying custumer-api

#### Deploying inventory-api