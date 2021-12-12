FROM quay.io/centos/centos:stream8

USER root

WORKDIR /work/

RUN chown 1001 /work 
RUN chmod "g+rwX" /work 
RUN chown 1001:root /work
RUN curl https://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-sender-3.0.9-1.el7.x86_64.rpm --output /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm 
RUN yum update -y
RUN yum install /tmp/zabbix-sender-3.0.9-1.el7.x86_64.rpm -y
RUN yum clean all

COPY --chown=1001:root target/*-runner /work/application

EXPOSE 8080

USER 1001

CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
