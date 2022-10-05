---
layout: post
title: Consistent Development Environments using Containers
date:   2022-10-04 20:57:00
---

One of the most common concerns we dealt with while I was interning at Snow College's IT department was making sure onboarding future interns would be as easy as possible. We spent quite a few meetings talking about technical standards we'd follow and how they would be taught. Another aspect of making sure onboarding would be easy was standardizing development environments. Making sure you have all the right tools installed and configured correctly can be difficult when you've barely just started developing apps for real. I remember being occasionally blocked from making progress during my first few weeks with the IT department simply because I didn't yet understand what I needed to make everything run correctly.

Luckily, we found a solution that would limit the amount of time new hires would spend setting up their environment and the time more experienced members of the team would have to spend helping them. The solution was remote development containers. Containers are great for ensuring something will run the same no matter where you run it. That consistency makes them great for setting up development environments, especially if the final app is being containerized too, which we regularly did at the IT department.

We all used VS Code as our IDEs when I worked at the IT department and it was a great tool for setting up development containers, so I'll be outlining how to do so in this article. te first step is to install the "Remote Development" extension for VS Code, which will allow you to attach VS Code to your development container. The next is to go to the top level of your project's directory and create a folder called ".devcontainer" and then add three files, "devcontainer.json", "Dockerfile", and "docker-compose.yml". First, I'll show you the Dockerfile I used for my last project at the IT department, which used Python and React:

```Dockerfile
FROM almalinux:8

RUN dnf -y install \
      https://download.oracle.com/otn_software/linux/instantclient/213000/oracle-instantclient-basic-21.3.0.0.0-1.el8.x86_64.rpm \
      openssh-clients \
      tar \
      gzip \
      git \
      nodejs \
      python38 python38-pip \
      gcc \
      python38-devel \
      openldap-devel && \
    rm -rf /var/cache/yum && \
    npm i -g n && \
    n stable

  
RUN groupadd --gid 1000 developer \
    && useradd --uid 1000 --gid 1000 -m developer \
    && chown -R developer. /home/developer

USER developer

RUN python3 -m venv /home/developer/.virtualenvs/portal \
  && source /home/developer/.virtualenvs/portal/bin/activate \
  && pip install --upgrade pip

WORKDIR /app

```

You can see from the first line that the development container being made here uses AlmaLinux as a base. You can then see on the second line all of the packages necessary for development being installed. The third and fourth lines create a user and set up permissions for them. The fifth line is  Python project-specific and sets up a venv for packages within the container.

After setting up your Dockerfile, you can move on to "docker-compose.yml". Again, I'll show the one I used for my last project:

```yml
version: '3'

services:
  portal:
    container_name: portal
    build: 
      context: .
    user: developer
    privileged: true
    volumes:
      - developer_home:/home/developer
      - ../:/app/:z
      - $HOME/.ssh:/home/developer/.ssh:z
      - ./.bashrc:/home/developer/.bashrc:z
      - ./.bash-git-prompt:/home/developer/.bash-git-prompt:z
    env_file:
      - .env
      
    command: >
      sh -c "
        cd api
        source /home/developer/.virtualenvs/portal/bin/activate
        pip install -r requirements.txt
        uvicorn src.main:app --host 0.0.0.0 --reload &
        cd ..
        cd client
        npm i
        npm start
      "
    ports:
      - "8080:3000"
      - "5000:8000"
    networks:
      shared:

  portal_db:
    image: postgres:latest
    container_name: portal_db
    environment:
      - POSTGRES_DB=Portal
      - POSTGRES_USER=PortalUser
      - POSTGRES_PASSWORD=testpsswd1!
    volumes: 
      - ../api/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    ports:
      - 5432:5432
    networks: 
      shared:

  portal_test_db:
    image: postgres:latest
    container_name: portal_test_db
    environment:
      - POSTGRES_DB=TestPortal
      - POSTGRES_USER=TestPortalUser
      - POSTGRES_PASSWORD=testpsswd1!
    volumes: 
      - ../api/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    ports:
      - 5433:5432
    networks: 
      shared:

volumes:
  developer_home:

networks:
  shared:
```

This dockerfile sets up the dev container, along with containerized instances of postgres for development and testing. You'll notice some stuff going on with the "volumes" section of the "portal" (i.e dev) container. The first volume mounts a home directory for the developer user, the second mounts the project directory, and the third mounts the .ssh directory from your computer into the container. The last two load in configurations for bash and its git cli (these configurations were added by my supervisor, not me).
Next, the "command" line starts up the Python backend api and the React frontend client. Lastly, some ports the backend and frontend use are mapped to ports on the machine running the container. I won't be talking about the Postgres containers since that's regular Docker stuff.

Lastly, lets look at "devcontainer.json"

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.191.0/containers/docker-from-docker-compose
{
	"name": "Docker from Docker Compose",
	"dockerComposeFile": "./docker-compose.yml",
	"service": "portal",
  	"shutdownAction": "stopCompose",
	"workspaceFolder": "/app/",

	// Use this environment variable if you need to bind mount your local source code into a new container.
	"remoteEnv": {
		"LOCAL_WORKSPACE_FOLDER": "${localWorkspaceFolder}"
	},
	// Set *default* container specific settings.json values on container create.
	"settings": {},

	// Add the IDs of extensions you want installed when the container is created.
	"extensions": [
		"irongeek.vscode-env",
		"ms-python.python",
		"esbenp.prettier-vscode",
		"dsznajder.es7-react-js-snippets"
	],

	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	"forwardPorts": [],

	// Use 'postCreateCommand' to run commands after the container is created.
	// "postCreateCommand": "code api",

	// Comment out connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
	"remoteUser": "developer",
  // "initializeCommand": "export GID=$(id -g)",
  // "updateRemoteUserUID": true
}
```

This file, as you can probably tell from the comments, was created by Microsoft. It does things like let VS Code know where your docker-compose.yml file is, which service in the docker-compose.yml file encapsulates your dev container, how to shut down the dev container, what VS Code extensions you want in the dev container, and so on. It should be noted that this file can be configured to use just a Dockerfile instead of a whole docker-compose.yml file, but the latter method gives you a lot more to work with.

To start up the dev container, open the project in VS Code, click on the blue pair of angle brackets in the bottom-left corner of the window, and select "Reopen in Container" on the command menu. That's it.

With all this set up, you can have multiple developers working on the same project using the exact same development environment. You can even check in the .devcontainer/ folder into source control so that new developers can simply check out the project and get started right away. 