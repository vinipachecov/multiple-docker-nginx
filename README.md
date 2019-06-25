# Docker and Kubernets course with Stephen Grinder

# About
This repo was created due to the section 8 to 11 of Stephen Grinder course on Docker and Kubernetes.
This section is related to creating a multi container application to give the student a taste of how a project with multiple containers can be handled in AWS ECS using Travis CI/CD.

Useful Links: 
 - <a href="https://www.udemy.com/docker-and-kubernetes-the-complete-guide">Stephen Course on Docker and Kubernetes</a>
 - <a href="https://jsonformatter.curiousconcept.com/">JSON Validator</a> 

The app is a fibonacci calculator, saving indexes in a cache service. The stack is the following:

* Redis (cache)
* React (frontend)
* Nodejs (worker and server)
* Postgres (database)
* Nginx (web server gateway)

What the instructor is basically helping us to understand is the whole workflow of a more complex application, so what will happen is the following:

## Node.js
We have 2 Node.js Apps:

- worker
- server

The api is going to handle if we are going to get value from db (or redis) or if we are going to send value to the worker to calculate.

The server provides api endpoints to get values from the db and an endpoit to evaluate if the requested value will be calculated or will be sent to the worker.


## React
React here is just a overenginnering app because is just a form that post something. The idea here is only about showing a use case regardless if it is a good use case of react.


## Nginx

Nginx will proxy the request finding the propper container to handle the request. To do this we end up overwriting the default configuration by a custom one.


## Setting up Dockerfiles
To get best experience setting up development Dockerfiles it is usefull to use the notation dev.Dockerfile so the code editor will still recognize as a Dockerfile. Using this naming convention gives us all auto complete stuff from our code editors.

### Dockerfile base model

Node apps:
```docker
FROM node:alpine
WORKDIR "/app"
COPY package.json ./
RUN npm i
COPY . .
CMD ["npm", "run", "dev"]
```

Here we are:
1. getting node:alpine docker image
2. Setting workdir to /app dir
3. copying package.json
4. installing app
5. copying other files
6. Running the container

All other dockerfiles are mostly equal, just changing the start command due to the package.json scripts(i.e react app).

### Nginx Dockerfile

Now, lets check nginx as this is the most different dockerfile:
```docker
FROM nginx
COPY default.conf /etc/nginx/conf.d/default.conf
```

Here we have:

1. Using nginx latest image from dockerhub
2. replacing the default.conf file with a custom one.

Here is our nginx configuration:

```
upstream client {
  server client:3000;  
}

upstream api {
  server api:5000;
}

server {
  listen 80;

  location / {
    proxy_pass http://client;
  }

  location /sockjs-node {
    proxy_pass http://client;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
  }


  location /api {
    rewrite /api/(.*) /$1 break;
    proxy_pass http://api;
  }
}
```
#### Upstreams
Well, we can see two upstream definitions. Upstreams are naming definitions to a specific host. Here we have client and api because the docker-compose file has a setup for both services as api and client.<br/>

As we now react is running on port 3000 we have <strong>client:3000</strong> and as the api is running on port 5000 we have <strong>api:5000</strong> so later we can only refer api and client withouth the port mapping

#### Server
The server configuration here is not about the server folder but how nginx will handle requests into a specific set of incoming  <strong>request paths</strong>.

What does that mean is:
- / path will proxy to the client (i.e, react app)
- /sockjs-node will handle react development server socket
- /api will proxy to the api(i.e, server), but will also rewrite the request url, removing the /api/ prefix. If we don't do this the server api will not be able to handle /api/values endpoit as it does not have such endpoint handled, leading to a 404 error.

### Travis CI

Let's check Travis configs:
```yml
sudo: required
services:
  - docker

before_install:
  - docker build -t vinipachecov/react-test -f ./client/dev.Dockerfile ./client  

script:
  - docker run vinipachecov/react-test npm run ci-test -- --coverage

# production version
# of every app
after_success:
  - docker build -t vinipachecov/multi-client ./client
  - docker build -t vinipachecov/multi-nginx ./nginx
  - docker build -t vinipachecov/multi-server ./server
  - docker build -t vinipachecov/multi-worker ./worker
  # Log in to the docker CLI
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_ID" --password-stdin
  - docker push vinipachecov/multi-client
  - docker push vinipachecov/multi-nginx
  - docker push vinipachecov/multi-server
  - docker push vinipachecov/multi-worker  
```
Sudo tag here is for superuser requirements.
The only service we are using here is Docker.

#### before_install
Here we've setup the tests pipelines. As we only have tests in our react app we only have a single line here.
ci-test here is equal to "CI=true react-scripts test" which has a CI variable so Jest testing framework can end properly after all tests.

### Dockerrun AWS

Elastic Beanstalk doesn't handle multiple containers unless we gave instructions of how to handle each of them through AWS ECS(Elastic Container Service).
To give those instructions we create a json with the configuration for our application to run.


```json
{
  "AWSEBDockerrunVersion": 2,  
  "containerDefinitions": [
    {
      "name": "client",
      "image": "vinipachecov/multi-client",
      "hostname": "client",
      "essential": false      
    },
    {
      "name": "server",
      "image": "vinipachecov/multi-server",
      "hostname": "api",
      "essential": false
    },
    {
      "name": "worker",
      "image": "vinipachecov/multi-worker",
      "hostname": "worker",
      "essential": false
    },
    {
      "name": "nginx",
      "image": "vinipachecov/multi-nginx",
      "hostname": "nginx",      
      "essential": true,
      "portMappings": [
        {
          "hostPort": 80,
          "containerPort": 80
        }
      ],
      "links": ["client", "server"]
    }
  ]
}
```


- Create a services
- Create a VPC
- GIve VPC the correct inbound rules for access 
- give services access to the vpc
- set env variables to aws containers