## A CD pipeline using ansible, docker and jenkins.

---
Here I used two ansible playbooks. One to create a docker image by pulling a github repository [link](https://github.com/j4jewel/ansible-CD) (which contains Dockerfile and web_content directory) The plabook will clone the github repository and then make a docker image with that particular Dockerfile. The docker image tags will be given according to the tags given in github commits. Suppose the developer makes commit and tag it as 2,3,4 like that in future commits. The playbook will create a docker image called j4jewel/ansible:(2,3,4 according to the tags given by developer) It will also add a latest tag to the image and push it to the dockerhub. https://hub.docker.com/r/j4jewel/ansiblecd 

The image build will only happen when a change is occured in the repository no matter how many times you manually run the playbook.

The second playbook is to download the latest docker image and run a docker container on a different server(Production)

By using Jenkins we can create a CD pipeline. We should create two jobs the first one to execute the build playbook and another job to deploy the container. The sourcecode of the first job should be set as the github repo (https://github.com/j4jewel/ansible-CD.git) set cron (POll SCM) as requirement.
In build area give the first playbook build.yml and create the second job and in build_trigger, set it as after the first project and give the deploy.yml playbook in the build section. 
---

#### build.yml
```sh
---

 - name: "Ansible CD for building Docker Image"
   hosts: localhost
   become: yes
   vars:
     git_url: https://github.com/j4jewel/ansible-CD.git
     clone_dir: /var/image_content
   tasks:

     - name: "Installing git"
       yum:
         name: git
         state: present

     - name: "Creating clone directory"
       file:
         path: "{{ clone_dir }}"
         state: directory

     - name: "Cloning git repository"
       git:
         repo: "{{ git_url }}"
         dest: "{{ clone_dir }}"
       register: repo_status

     - name: "Getting the latest tag from git"
       shell: "git describe --tags"
       args:
         chdir: "{{ clone_dir }}"
       register: latest_tag

     - debug: var=latest_tag.stdout

     - name: "Installing Docker and pip"
       yum:
         name: python-pip
         state: present

     - name: "Installing docker-py"
       pip: name=docker-py

     - name: Docker Login"
       docker_login:
         username: j4jewel
         password: ***DockerHub_Password***
         email: jwlalias95@gmail.com

     - name: "Building the DockerImage"
       docker_image:
         path: "{{ clone_dir }}"
         name: "j4jewel/ansiblecd"
         tag: "{{ item }}"
         push: yes
         force: yes
       with_items:
         - latest
         - "{{ latest_tag.stdout }}"
       when: repo_status.changed

     - name: "Removing the images"
       docker_image:
         state: absent
         name: "j4jewel/ansiblecd"
         tag: "{{ item }}"
       with_items:
         - latest
         - "{{ latest_tag.stdout }}"

```
#### deploy.yml
```sh
---

 - name: "Creating Container"
   hosts: prod
   become: yes
   tasks:

     - name: "Installing Docker and pip"
       yum:
         name: python-pip
         state: present

     - name: "Installing docker-py"
       pip: name=docker-py


     - name: "Running Docker Container"
       docker_container:
         name: "webserver"
         recreate: true
         pull: yes
         image: "j4jewel/ansiblecd:latest"
         published_ports:
          - "8080:80"

```



