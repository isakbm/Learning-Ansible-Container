# Learning Ansible-Container

## Steps to get Ansible Container working

We are following [this](https://docs.ansible.com/ansible-container/installation.html) guide for instalation and an example setting up a flask server.
Some of the steps are tricky due to incompatibilities with newer version of pip, etc

- [Installation](#installation)
- [Getting Hello World Server Started](#getting-hello-world-server-started)

## Installation

I did this on ubuntu xenial 16.04.4 LTS
with the workon wrapper for virtualenv installed


```
$ virtualenv --version
15.0.1
$ virtualenv ansible-container
...
Installing setuptools, pkg_resources, pip, wheel...done.
$ workon ansible-container
$ pip install --upgrade pip==9.0.3
...
$ pip install ansible-container[docker]
$ pip install --upgrade docker==2.7.0
```

Once that is all done, you should have your ansible-container virtual environment ready to go.

Now let's proceed with getting the Hello World flask server up and running.

# Getting Hello World Server Started

You will find all the necessary file content from [this](https://docs.ansible.com/ansible-container/getting_started.html) link.

First set up the project directory and initialize the ansible-container

```
$ mkdir hello-world
$ cd hello-world
$ ansible-container init
Ansible Container initialized.
```

Then make a requirements.txt file (to be used by python from inside the container later)
```
$ cat hello-world/requirements.txt

Flask=0.12.2
gunicorn=19.7.1
```

And the hello world flask script should also go in the hello-world directory, the file reads

```py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
```

We also need a role for the flask service, note that these commands are run inside the hello-world directory

```
$ mkdir roles
$ cd roles
$ ansible-galaxy init flask
- flask was created successfully
```

Now, the ansible recipe. First navigate to where it should be placed. That is, assuming you are in the hello-world directory do

```
$ cd roles/flask/tasks
```

You will see that there is already a *main.yml* file there. Replace it's contents with

```yaml
---
- name: Install dumb init
  get_url:
    dest: /usr/bin/dumb-init
    url: https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64
    mode: 0775
    validate_certs: no
- name: Install epel
  yum:
    name: epel-release
    state: present
    disable_gpg_check: yes
- name: Install pip
  yum:
    name: python2-pip
    state: present
    disable_gpg_check: yes
- name: Create flask user
  user:
    name: flask
    uid: 1000
    group: root
    state: present
    createhome: yes
    home: "/app"
- name: Copy source into container
  synchronize:
    src: "/src/"
    dest: "/app"
  remote_user: flask
- name: Install Python dependencies
  pip:
    requirements: /app/requirements.txt
```

Finally, navigating up again to the hello-world directory, modify the container.yml file to contain

```yaml
version: "2"
settings:
  conductor:
    base: centos:7
  project_name: hello-world
  k8s_namespace:
     name: hello-world
     description: Simple Demo
     display_name: Hello, World
services:
  flask:
    from: centos:7
    roles:
      - flask
    entrypoint: /usr/bin/dumb-init
    command: gunicorn -w 4 -b 0.0.0.0:4000 helloworld:app
    ports:
      - 4000:4000
    working_dir: /app
    user: flask
registries: {}
```

And that's pretty much it, now build the container, and run it.

```
$ ansible-container build
...

$ ansible-container run
...

$ curl http://127.0.0.1:4000/
Hello, World!
```

You should also see it listed by running
```
$ docker ps
... 
```
