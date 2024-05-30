# Introduction

We want to use Ansible to add everything needed for our server to work.

## Inventories

First step is to connect ansible to our server by giving the address and the key.

### setup.yml

all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ./id_rsa
  children:
    prod:
      hosts: prosper.playoust.takima.cloud

# Deploy your App

## Document your playbook

We want to create roles in our playbook. The roles take care of each task to do the actions needed.

### Docker

This one is to make sure we download docker and its dependencies and make sure it works

`ansible-galaxy init roles/docker`

- name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker

### Network

This one is to create the network to connect the different contenair

`ansible-galaxy init roles/network`

name: Create a network
  community.docker.docker_network:
    name: prosp_network

### Database

This one run the contenair database and put it inside the network

`ansible-galaxy init roles/database`

name: Run database
  docker_container:
    name: database
    image: pepsouille0/database:1.0
    networks: 
      - name: prosp_network

### Httpd 

This one run the container httpd, open the ports 80:80 and put it inside the network

`ansible-galaxy init roles/httpd`

name: Run HTTPD
  docker_container:
    name: httpd
    image: pepsouille0/discover-docker-httpd:1.0
    networks: 
      - name: prosp_network
    ports: 
      - "80:80"

### Backend

This one run the contenair backend, put it inside the network and open the ports 8080:8080

`ansible-galaxy init roles/backend`

name: Run backend
  docker_container:
    name: backend_db
    image: pepsouille0/discover-docker-backend:1.0
    networks: 
      - name: prosp_network
    ports: 
      - "8080:8080"


We also need to think about the names of the contenair that were inside the .conf file.

When everything done correctly, we can then go on our website to interact with the data.

# Continuous Deployment

To have a continuous deployment, we need a new workflow. 
This one download ansible and run the playbook to put the application in production

name: Deploy Application
 
on:
  workflow_run:
    workflows: ["CI devops 2024"]
    types:
      - completed
 
jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
 
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
 
      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.5.3
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
 
      - name: Install Ansible
        run: |
          sudo apt-get update
          sudo apt-get install -y ansible
 
      - name: Disable host key checking
        run: echo "ANSIBLE_HOST_KEY_CHECKING=False" >> $GITHUB_ENV
 
      - name: Run deployment playbook
        run: |
          ansible-playbook -i ../../Ansible/inventories/setup.yml ../../Ansible/playbook.yml
        env:
          ANSIBLE_HOST_KEY_CHECKING: False
