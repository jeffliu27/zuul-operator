FROM quay.io/operator-framework/ansible-operator:v0.6.0

COPY watches.yaml ${HOME}/watches.yaml

COPY ansible/zuul.yaml ${HOME}/zuul.yaml
COPY ansible/group_vars/ ${HOME}/group_vars/
COPY ansible/roles/ ${HOME}/roles/

COPY build/uid_entrypoint.sh /uid_entrypoint

USER root

RUN yum install -y telnet