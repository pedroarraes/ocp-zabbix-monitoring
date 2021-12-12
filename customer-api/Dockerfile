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
