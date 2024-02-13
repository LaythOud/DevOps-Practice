# DevOps Practice

## Test Description

### Test assignment A: Docker and Microservices
**Objective:** To assess the candidate’s ability to work with Docker and microservices architecture.

**Task Description:**
- This app includes App (PHP), MySQL and Redis services.
- Containerize these services using Docker. Each service should have its own Dockerfile.
- Write a docker-compose.yml file to orchestrate the services.
- Ensure that the services can communicate with each other and are scalable.
- Document the steps to build and run the containers, and any assumptions made in the process.

**Evaluation Criteria:**
- Understanding of Docker concepts and containerization.
- Ability to set up a microservices architecture.
- Quality of the Dockerfiles and docker-compose configurations.
- Documentation and clarity of instructions.

### Test assignment B: Continuous Integration/Continuous Deployment (CI/CD)
**Objective:** To evaluate the candidate's experience with CI/CD pipelines and version control systems.

**Task Description:**
- Using the output from Test A. 
- Create a CI/CD pipeline using a tool like Jenkins, GitLab CI, or GitHub Actions (The pipeline should include stages for building, testing, and deploying the application).
- Document the CI/CD setup and provide instructions on how the pipeline works, including how to trigger builds and deployments.

**Evaluation Criteria:**
- Proficiency in using version control (Git).
- Ability to set up and configure CI/CD pipelines.
- Understanding of automated testing and its integration into CI/CD.
- Documentation quality and the ability to explain the CI/CD process.

### Test assignment C: Deployment to a Cloud or Server Environment
**Objective:**  To assess the candidate’s ability to deploy applications to a cloud or server environment, integrating the work done in Test B.

**Task Description:**
- Take the CI/CD pipeline and the web application developed in Test B (If you are not familiar with CI/CD, you may use output from Test A instead).
- Extend the CI/CD pipeline to include a deployment stage that automatically deploys the application to a cloud provider (AWS, Azure, GCP) or a server environment of your choice.
- Ensure the deployment includes necessary environment configurations (like virtual networks, security groups, etc.) and is scalable.
- Implement basic monitoring and logging for the application in the cloud/server environment.
- Document the deployment process, including any scripts or configuration files used, and instructions on how to access the deployed application.

## Overview

This repository contains a Laravel project integrated with MySQL and Redis, each encapsulated in Docker containers. The project is managed through a GitLab pipeline with five stages: build, test, release, pre-deploy, and deploy. The pre-deploy stage involves Ansible code for server configuration.

### Deployment Environment

The deployment environment uses Azure VMs for setup. Terraform is not used in this project for cloud resource access.

I have deployed the Ansible image to the container registry, intentionally excluding it from the build stage to enhance security measures. Kindly generate your own image by following the provided steps.

This project offers the capability to integrate an SSL layer into the Laravel application using Let's Encrypt. This can be achieved by adding `apache-ssl.conf` to the Laravel Docker file instead of `apache.conf`, ensuring the correct path to the Let's Encrypt files and opening port 443. It is crucial to emphasize that activating this feature requires the provision of a valid domain name.

## Docker Setup

### Dockerfiles

#### Laravel Dockerfile (app/Dockerfile)

```dockerfile
FROM php:8.2-apache

# Copy composer.lock and composer.json to the working directory
COPY composer.lock composer.json /var/www/html/

# Install any needed packages
RUN apt-get update && apt-get install -y \
    libzip-dev \
    unzip

# Install PHP extensions
RUN docker-php-ext-install pdo pdo_mysql zip

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Install Laravel dependencies
RUN composer install --no-scripts --no-autoloader

# Copy the rest of the application code
COPY . /var/www/html

# Generate the optimized autoloader
RUN composer dump-autoload --optimize

# Set the correct permissions
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

COPY cicd/apache.conf /etc/apache2/sites-available/

RUN a2ensite apache
RUN a2dissite 000-default.conf
```

#### MySQL Dockerfile (Dockerfile.database)

```dockerfile
FROM mysql/mysql-server:8.0
```

#### Redis Dockerfile (Dockerfile.redis)

```dockerfile
FROM redis:alpine

VOLUME /data

HEALTHCHECK --interval=5s --timeout=5s --retries=3 \
             CMD redis-cli ping || exit 1
```
#### Ansible Dockerfile (Ansible/Dockerfile)


```dockerfile
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


```

### Docker Compose File (docker-compose.yml)

```yaml
services:
    app:
        container_name: staytus-app
        build:
            context: ./App
            dockerfile: Dockerfile
        extra_hosts:
            - 'host.docker.internal:host-gateway'
        ports:
            - '${APP_PORT:-80}:80'
        networks:
            - sail
        depends_on:
            - mysql
            - redis
    mysql:
        container_name: staytus-mysql
        build:
            context: .
            dockerfile: Dockerfile.database
        ports:
            - '${FORWARD_DB_PORT:-3306}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${MYSQL_ROOT_PASSWORD}'
            MYSQL_ROOT_HOST: '${MYSQL_ROOT_HOST:-%}'
            MYSQL_DATABASE: '${MYSQL_DATABASE}'
            MYSQL_PASSWORD: '${MYSQL_PASSWORD}'
            MYSQL_USERNAME: '${MYSQL_USERNAME}'
        # volumes:
        # - 'sail-mysql:/var/lib/mysql'
        #     - './vendor/laravel/sail/database/mysql/create-testing-database.sh:/docker-entrypoint-initdb.d/10-create-testing-database.sh'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - mysqladmin
                - ping
                - '-p${MYSQL_ROOT_PASSWORD}'
            retries: 3
            timeout: 5s
        restart: unless-stopped

    redis:
        container_name: staytus-redis
        build:
            context: .
            dockerfile: Dockerfile.redis
        ports:
            - '${FORWARD_REDIS_PORT:-6379}:6379'
        volumes:
            - 'sail-redis:/data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - redis-cli
                - ping
            retries: 3
            timeout: 5s
        restart: unless-stopped

networks:
    sail:
        driver: bridge

volumes:
    sail-mysql:
        driver: local
    sail-redis:
        driver: local
```

## GitLab Pipeline

### Stages

1. **Build**: Building Docker images.
2. **Test**: Running tests on the Laravel application, Mysql database and redis.
3. **Release**: Preparing the Docker images for deployment.
4. **Pre-Deploy**: Configuring servers using Ansible.
5. **Deploy**: Deploying the application.

### .gitlab-ci.yml

```yaml
---
---

stages:
  - build
  - test
  - release
  - pre-deploy
  - deploy

variables:
  IMAGE_PREFIX: $CI_REGISTRY/laythoud/staytus-assessment
  CONTAINER_TEST_TAG: $CI_COMMIT_REF_SLUG
  CONTAINER_RELEASE_TAG: latest


default:
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  image: docker:20.10.16
  stage: build
  services:
    - docker:20.10.16-dind
  script:
    - docker build --pull -t $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG ./App
    - docker build --pull -t $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG -f Dockerfile.database .
    - docker build --pull -t $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG -f Dockerfile.redis .
    - docker push $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG 
    - docker push $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG 
    - docker push $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG
  timeout: 5 minutes
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH


test-mysql:
  image: docker:20.10.16
  stage: test
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG 
    - docker run -d --name=mysql-test -e MYSQL_ALLOW_EMPTY_PASSWORD=1 -p 3306:3306 $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG
    - sleep 20 && docker exec mysql-test mysql
  timeout: 2 minutes
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

test-redis:
  image: docker:20.10.16
  stage: test
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG
    - docker run -d --name=redis-test -p 6379:6379 $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG
    - sleep 5 && docker exec redis-test redis-cli
  timeout: 2 minutes
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

test-app:
  image: docker:20.10.16
  stage: test
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG 
    - docker run -d --name=app-test -e APP_KEY=$APP_KEY -e DB_CONNECTION=mysql 
      -e DB_HOST=$MYSQL_ROOT_HOST -e DB_PORT=3306 -e DB_DATABASE=$MYSQL_DATABASE 
      -e DB_USERNAME=$MYSQL_USERNAME -e DB_PASSWORD=$MYSQL_PASSWORD 
      -e REDIS_HOST=$REDIS_HOST -e REDIS_PASSWORD=$REDIS_PASSWORD -e REDIS_PORT=6379 
      -p 80:80 $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG
    - sleep 5 && docker exec  app-test /bin/bash test.sh
  timeout: 2 minutes
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

release-image:
  image: docker:20.10.16
  stage: release
  services:
    - docker:20.10.16-dind
  script:
    - docker pull $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG 
    - docker pull $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG 
    - docker pull $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG
    - docker tag $IMAGE_PREFIX/staytus-app:$CONTAINER_TEST_TAG $IMAGE_PREFIX/staytus-app:$CONTAINER_RELEASE_TAG
    - docker tag $IMAGE_PREFIX/staytus-database:$CONTAINER_TEST_TAG $IMAGE_PREFIX/staytus-database:$CONTAINER_RELEASE_TAG
    - docker tag $IMAGE_PREFIX/staytus-redis:$CONTAINER_TEST_TAG $IMAGE_PREFIX/staytus-redis:$CONTAINER_RELEASE_TAG
    - docker push $IMAGE_PREFIX/staytus-app:$CONTAINER_RELEASE_TAG 
    - docker push $IMAGE_PREFIX/staytus-database:$CONTAINER_RELEASE_TAG 
    - docker push $IMAGE_PREFIX/staytus-redis:$CONTAINER_RELEASE_TAG
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

pre-deploy:
  image: $IMAGE_PREFIX/staytus-ansiable:latest
  stage: pre-deploy
  before_script: []
  script:
    - ansible-playbook -i Ansible/hosts Ansible/deploy.yml -e ENV=PRODUCTION --vault-password-file /root/.ansible_vault 
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  tags:
    - deploy
    - prod-runner

deploy:
  stage: deploy
  before_script: []
  script:
    - /bin/bash deploy.sh
  allow_failure: false
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  tags:
    - deploy
    - prod-runner
```

## Ansible Setup

### Ansible Directory Structure

```
Ansible/
|-- vars/
|   |-- vars.prod.yml
|-- hosts
|-- deploy.yml
|-- .ansible_vault
```

### Ansible Configuration Files

#### Ansible/vars/vars.prod.yml

```yaml
---
GITLAB_REGISTRY:
GITLAB_REGISTRY_TOKEN:
GITLAB_RUNNER_TAGS:
```

#### Ansible/hosts

```ini
[main]
server ansible_host= ansible_user= ansible_port= ansible_ssh_private_key_file=
```

#### Ansible/deploy.yml

```yaml
---
---

- name: Download and Install GitLab Runner 
  hosts: main
  gather_facts: false
  become: true
        
  tasks:
    - name: Update and upgrade all packages to the latest version
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        cache_valid_time: 3600

    - name: Install dependencies
      apt:
        name:
          - git
          - wget

    - name: Download GitLab Runner script
      shell: |
        wget -qO- https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

    - name: Install GitLab Runner
      apt:
        name:
          - gitlab-runner


- name: Register a GitLab Runner
  hosts: main
  gather_facts: false
  become: true

  tasks:
    - name: Include production variables
      include_vars:
        file: "vars.prod.yml"
      when: ENV == "PRODUCTION"

    - name: Register GitLab Runner
      shell: |
          sudo gitlab-runner register \
            --non-interactive \
            --url {{ GITLAB_REGISTRY }} \
            --registration-token {{ GITLAB_REGISTRY_TOKEN }} \
            --executor "shell" \
            --description "Production Runner" \
            --tag-list {{ GITLAB_RUNNER_TAGS }} \

    - name: Start GitLab Runner service
      service:
        name: gitlab-runner
        state: started
        enabled: yes



- name: Install Docker and Docker Compose
  hosts: main
  become: true
  vars:
    arch_mapping:  # Map ansible architecture {{ ansible_architecture }} names to Docker's architecture names
      x86_64: amd64
      aarch64: arm64
  
  tasks:
    - name: Include production variables
      include_vars:
        file: "vars.prod.yml"
      when: ENV == "PRODUCTION"

    - name: Update and upgrade all packages to the latest version
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        cache_valid_time: 3600

    - name: Install required packages
      ansible.builtin.apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - software-properties-common

    - name: Create directory for Docker's GPG key
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker's official GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        keyring: /etc/apt/keyrings/docker.gpg
        state: present
        
    - name: Print architecture variables
      ansible.builtin.debug:
        msg: "Architecture: {{ ansible_architecture }}, Codename: {{ ansible_lsb.codename }}"

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: >-
          deb [arch={{ arch_mapping[ansible_architecture] | default(ansible_architecture) }}
          signed-by=/etc/apt/keyrings/docker.gpg]
          https://download.docker.com/linux/ubuntu {{ ansible_lsb.codename }} stable
        filename: docker
        state: present

    - name: Install Docker and related packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
        update_cache: true
      loop:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - docker-buildx-plugin
        - docker-compose-plugin

    - name: Add Docker group
      ansible.builtin.group:
        name: docker
        state: present

    - name: Add user to Docker group
      ansible.builtin.user:
        name: "{{ item }}"
        groups: docker
        append: true
      loop:
        - "{{ ansible_user }}"
        - gitlab-runner

    - name: Enable and start Docker services
      ansible.builtin.systemd:
        name: "{{ item }}"
        enabled: true
        state: started
      loop:
        - docker.service
        - containerd.service
```

### Encrypt Sensitive Data with Ansible Vault

To encrypt sensitive data in Ansible, you can use Ansible Vault. Follow these steps:

1. Create an encrypted file for sensitive data:

   ```bash
   ansible-vault encrypt --vault-password-file=Ansible/.ansible_vault Ansible/vars/vars.prod.yml
   ```

2. Add sensitive data to the file and save it.

3. Save the vault password securely.

To edit the encrypted file:

```bash
   ansible-vault edit --vault-password-file=Ansible/.ansible_vault Ansible/vars/vars.prod.yml
```

To run an Ansible playbook with an encrypted file:

```bash
ansible-playbook -i Ansible/hosts Ansible/deploy.yml -e ENV=PRODUCTION --vault-password-file=Ansible/.ansible_vault                                        
```

## Conclusion

This README provides a comprehensive guide for setting up a Laravel project with Docker containers, GitLab pipeline, and Ansible for server configuration. Ensure to replace placeholder values with your specific configurations. Follow the steps carefully for secure configuration management using Ansible Vault.