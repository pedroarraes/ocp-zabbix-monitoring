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