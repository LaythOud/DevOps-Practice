FROM ubuntu:22.04

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update && apt-get install sshpass openssh-client software-properties-common python3-pip python3-dev libffi-dev libssl-dev python3-passlib -y
RUN pip3 install ansible
RUN pip3 install --upgrade pip && pip3 install --upgrade setuptools && pip3 install --upgrade cryptography pyOpenSSL ndg-httpsclient pyasn1
RUN pip3 install --upgrade urllib3[secure]

COPY .ansible_vault /root/.ansible_vault

ENV SSH_KEY ~/.ssh/key.pem
COPY ${SSH_KEY} /root/.ssh/key.pem

COPY hosts /etc/ansible/hosts

RUN chmod 0400 /root/.ansible_vault

COPY hosts Ansible/hosts
COPY vars/vars.prod.yml Ansible/vars/vars.prod.yml
COPY deploy.yml Ansible/deploy.yml

CMD ["/bin/bash"]
