# Monitoring OpenShift PODS with Ansible and Zabbix Sender 
This smart-start describes the creation  an OpenShift project to monitoring PODS using Zabbix Sender and Ansible

## Prerequisites
* OpenShift or generic kubernetes cluster
* OpenShift client (oc) or kubectl
* OpenJDK 11 or Graalvm-11
* Zabbix Server
* podman or docker

## Summary

* [Creating OpenShift Projects](#creating-openshift-projects)
  * [Monitoring API Project](#monitoring-api-project])
    * [Build ansible-agent4ocp](#build-ansible-agent4ocp)
    * [Testing ansible-agent4ocp](#testing-ansible-agent4ocp)
    * [Login in OCP public registry](#login-in-ocp-public-registry)
    * [Pushing image to OCP](#pushing-image-to-ocp)
  * [APIs project](#apis-project)
    * [Deploying custumer-api](#deploying-custumer-api)
    * [Deploying inventory-api](#deploying-inventory-api)
* [Configuring service account](#configuring-service-account)    
* [Scheduling OpenShift CronJobs](#scheduling-openshift-cronjobs) 
  * [Getting Service Account Token](#getting-service-account-token)
  * [Creating Ansible File as Config MAP for custumer-api](#creating-ansible-file-as-config-map-for-custumer-api)
  * [Creating Ansible File as Config MAP for inventory-api](#creating-ansible-file-as-config-map-for-inventory-api)
  * [Configuring CronJob for custumer-api](#configuring-cronjob-for-custumer-api)
  * [Configuring CronJob for inventory-api](#configuring-cronjobf-for-inventory-api)


## Creating OpenShift Projects
In this session we'll create an OpenShift projects to deploy apis example and CronJobs to get container metrics and send to Zabbix Server.

### Monitoring API Project
In this project we'll deploy ansible-agent4ocp to monitoring APIs PODs.

```bash
$ oc new-project apis-monitoring
```
```console
Now using project "apis-monitoring" on server "omitted".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.
```
#### Build ansible-agent4ocp
This is the Dockerfile for ansible-agent4ocp
```dockerfile
#base image
FROM quay.io/centos/centos:stream8 

USER root

#workdir folder
ENV HOME=/opt/scripts 

WORKDIR ${HOME}  

#CentoOS Stream extras repositories
RUN yum install epel-release -y && \ 
    #updating SO
    yum update -y && \ 
    yum install ansible.noarch -y && \ 
    #install ansible
    yum clean all && \
    #download and install OpenShift client
    curl https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz --output /tmp/oc.tar.gz && \ 
    tar xvzf /tmp/oc.tar.gz && \
    cp oc /usr/local/bin && \
    rm oc kubectl && \
    rm /tmp/oc.tar.gz && \
    #Granting permissions to folders and files
    mkdir -pv ${HOME} && \  
    mkdir -pv ${HOME}/.ansible/tmp && \
    mkdir -pv ${HOME}/.kube/ && \
    mkdir -pv ${HOME}/playbooks && \
    chown -R 1001:root ${HOME} &&  \
    chgrp -R 0 ${HOME} && \
    chmod -R g+rw ${HOME} 

#folder to save playbooks
VOLUME ${HOME}/playbooks 

USER 1001

#ansible example file
ADD example.yml ${HOME}/example.yml 
```
```bash
$ podman build ansible-agent4ocp/. --tag ansible-agent4ocp 
```
```console
STEP 1/8: FROM quay.io/centos/centos:stream8
STEP 2/8: USER root
omitted
3dae1ac7584c97e57296f674b42ac1886a88c180aec715860c8118712a81d683
```
#### Testing ansible-agent4ocp
```bash
$ podman run -it ansible-agent4ocp:latest ansible-playbook example.yml
```
```console
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [This is a ansible script hello-world] **************************************************************************************************************************************************************************************************

TASK [Hello Ansible] *************************************************************************************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ***********************************************************************************************************************************************************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
#### Login in OCP public registry
```bash
$ podman login -u $(oc whoami) -p $(oc whoami -t) <ocp-public-registry>
```
```console
Login Succeeded!
```
#### Pushing image to OCP
1. **Tag image**
```bash
$ podman tag localhost/ansible-agent4ocp <ocp-public-registry>/apis-monitoring/ansible-agent4ocp
```

2. **Push Image**
```bash
$ podman push <ocp-public-registry>/apis-monitoring/ansible-agent4ocp
``` 
```console
Getting image source signatures
Copying blob e7a4bda8f16d done  
omitted
Storing signatures
```
### APIs project
In this project we'll deploy example APIs.
```bash
$ oc new-project api
``` 
```console
Now using project "api" on server "omitted".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.

```
#### Deploying custumer-api

```dockerfile
FROM quay.io/centos/centos:stream8

USER root

WORKDIR /work/

RUN chown 1001 /work && \
    chmod "g+rwX" /work && \
    chown 1001:root /work && \
    #SO Update
    yum update -y && \
    #Zabbix Sender Install
    curl https://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-sender-3.0.9-1.el7.x86_64.rpm --output /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm && \ 
    yum install /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm -y && \
    yum clean all 

COPY --chown=1001:root target/*-runner /work/application

EXPOSE 8080

USER 1001

CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

1. **Deploying application in OCP**
```bash
$ oc new-app https://github.com/pedroarraes/ocp-zabbix-monitoring.git --context-dir=/customer-api --strategy=docker --name=customer-api
```
```console
--> Found Docker image dc28896 (5 weeks old) from quay.io for "quay.io/centos/centos:stream8"

    CentOS Stream 8 
    --------------- 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

omitted

    Run 'oc status' to view your app.
```
2. **Exposing application**
```bash
$ oc expose svc/customer-api
```
```console
route.route.openshift.io/customer-api exposed
``` 
3. **Testing api**
```bash
$ curl $(oc get route customer-api | awk 'FNR==2{print $2}')/hello
```
```console
Hello RESTEasy
```
4. **Testing Zabbix Sender**
```bash
$ oc get pods
```
```console
NAME                    READY     STATUS      RESTARTS   AGE
customer-api-1-build    0/1       Completed   0          12m
customer-api-1-c2rrx    1/1       Running     0          10m
customer-api-1-deploy   0/1       Completed   0          10m
```
```bash
$ oc rsh customer-api-1-c2rrx zabbix_sender
```
```console
zabbix_sender [23]: either '-c' or '-z' option must be specified
usage:
omitted
command terminated with exit code 1
```
#### Deploying inventory-api
```dockerfile
FROM quay.io/centos/centos:stream8

USER root

WORKDIR /work/

RUN chown 1001 /work && \
    chmod "g+rwX" /work && \
    chown 1001:root /work && \
    #SO Update
    yum update -y && \
    #Zabbix Sender Install
    curl https://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-sender-3.0.9-1.el7.x86_64.rpm --output /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm && \ 
    yum install /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm -y && \
    yum clean all 

COPY --chown=1001:root target/*-runner /work/application

EXPOSE 8080

USER 1001

CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

```bash
$ oc new-app https://github.com/pedroarraes/ocp-zabbix-monitoring.git --context-dir=/inventory-api --strategy=docker --name=inventory-api
```
```console
--> Found Docker image dc28896 (5 weeks old) from quay.io for "quay.io/centos/centos:stream8"

    CentOS Stream 8 
    --------------- 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

omitted

    Run 'oc status' to view your app.

```
2. **Exposing application**
```bash
$ oc expose svc/inventory-api
```
```console
route.route.openshift.io/inventory-api exposed
``` 
3. **Testing api**
```bash
$ curl $(oc get route inventory-api | awk 'FNR==2{print $2}')/hello
```
```console
Hello RESTEasy
```
4. **Testing Zabbix Sender**
```bash
$ oc get pods
```
```console
customer-api-1-build     0/1       Completed   0          35m
customer-api-1-c2rrx     1/1       Running     0          33m
customer-api-1-deploy    0/1       Completed   0          33m
inventory-api-1-deploy   0/1       Completed   0          2m19s
inventory-api-1-fswtb    1/1       Running     0          2m15s
inventory-api-3-build    0/1       Completed   0          4m46s

```
```bash
$ oc rsh inventory-api-1-fswtb zabbix_sender
```
```console
zabbix_sender [23]: either '-c' or '-z' option must be specified
usage:
omitted
command terminated with exit code 1
```

## Configuring service account
In this session we will create e configure service account permission to access OpenShift PODS, get metrics and send to Zabbix Server using Zabbix Sender.

1. **Creating service account**
```bash
$ oc create sa sa-apis-monitoring -n apis-monitoring
```
```console
serviceaccount/sa-apis-monitoring created
```

2. **Creating custom role binding to get and exec pods**
```bash
$ oc create role podview --verb=get,list,watch --resource=pods -n api
```
```console
role.rbac.authorization.k8s.io/podview created
```
```bash
$ oc create role podexec --verb=create --resource=pods/exec -n api
```
```console
role.rbac.authorization.k8s.io/podexec created
```
```bash
$ oc create role projectview --verb=get,list --resource=project -n api
```
```console
role.rbac.authorization.k8s.io/projectview created
```
3. **Adding policy to service account**
```bash
$ oc adm policy add-role-to-user podview system:serviceaccount:apis-monitoring:sa-apis-monitoring --role-namespace=api -n api
```
```console
role "podview" added: "system:serviceaccount:apis-monitoring:sa-apis-monitoring"
```

```bash
$ oc adm policy add-role-to-user podexec system:serviceaccount:apis-monitoring:sa-apis-monitoring --role-namespace=api -n api
```
```console
role "podexec" added: "system:serviceaccount:apis-monitoring:sa-apis-monitoring"
```

```bash
$ oc adm policy add-role-to-user projectview system:serviceaccount:apis-monitoring:sa-apis-monitoring --role-namespace=api -n api
```
```console
role "projectview" added: "system:serviceaccount:apis-monitoring:sa-apis-monitoring"
```
## Scheduling OpenShift CronJobs
In this session we'll scheduler OpenShift CronJobs to get metrics PODS and send to Zabbix Server

### Getting Service Account Token
This command is used to get secret value and will be used in Ansible Script.
```bash
$ oc describe secret $(oc describe sa sa-apis-monitoring -n apis-monitoring | awk  '{if(NR==8) print $2}') -n apis-monitoring | grep token | awk '{if(NR==3) print $2'}
```
```console
omitted
```

### Creating Ansible File as Config MAP to get free memory PODS
```yaml
- name: Get POD free memory
  hosts: localhost
  tasks:
  - name: OCP Autentication
    #Use the script at last session to take token
    shell: oc login --token=<omitted> --server=<omitted>
- name: Get PODS
  hosts: localhost
  tasks:
  - name: Go to API project
    shell: oc project api
  - name: Get PODs
    shell: oc get pods -n api | grep Running | awk {'print $1'}
    register: pods_list  
  - name: Get free memory
    shell: oc rsh {{ item }} zabbix_sender -vv -z <zabbix_server_host> -s <zabbix_registered_api>  -k free_memory -o $(oc rsh {{ item }} free | awk '{if(NR==2) print $4}')
    with_items: "{{ pods_list.stdout_lines }}"
```
```bash
$ oc create configmap free-memory --from-file=ansible-scripts/free-memory.yml -n apis-monitoring
```
```console
configmap/free-memory created
```
### Creating Ansible File as Config MAP to get used memory PODS
```yaml
- name: Get POD used memory
  hosts: localhost
  tasks:
  - name: OCP Autentication
    #Use the script at last session to take token
    shell: oc login --token=<omitted> --server=<omitted>
- name: Get PODS
  hosts: localhost
  tasks:
  - name: Go to API project
    shell: oc project api
  - name: Get PODs
    shell: oc get pods -n api | grep Running | awk {'print $1'}
    register: pods_list  
  - name: Get used memory
    shell: oc rsh {{ item }} zabbix_sender -vv -z <zabbix_server_host> -s <zabbix_registered_api>  -k free_memory -o $(oc rsh {{ item }} free | awk '{if(NR==2) print $3}')
    with_items: "{{ pods_list.stdout_lines }}"
```
```bash
$ oc create configmap used-memory --from-file=ansible-scripts/used-memory.yml -n apis-monitoring
```
```console
configmap/used-memory created
```
### Configuring CronJob for custumer-api

### Configuring CronJob for inventory-api
