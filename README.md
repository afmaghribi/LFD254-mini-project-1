This repository contains step by step to finish task `mini-project-1` of `CONTAINERS FOR DEVELOPERS AND QUALITY ASSURANCE (LFD254)` from Linux foundation training.

# Project Spec

→ given task to containerizing an application
→ make it as microservices
→ building images and then deploy the application using `docker-compose`
→ Before containerizing application, first we need to understand application stack before run it as microservices
→ vote ui → python
→ result ui → node.js
→ redis
→ db → postgresql
→ worker → .NET/Java
→ redis and potgres can use images from public repository. The rest app need to build manually
→ then create docker-compose spec to run the microservices

Before start, first clone the application repository

```
git clone https://github.com/LFD254/example-voting-app
```

## Prerequisite install docker

```
apt install docker.io docker-compose-v2
```

## Prerequisite setup harbor

- Install harbor offline installer 

```
wget https://github.com/goharbor/harbor/releases/download/v2.11.1/harbor-offline-installer-v2.11.1.tgz
tar -zxvf harbor-offline-installer-v2.11.1.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml // change to use desired port
```

- Add harbor as insecure registries

```
nano /etc/docker/daemon.json
{
  "insecure-registries": ["81.81.81.11:1337"]
}
```

## 1. Build a vote application (python)

- Create `Dockerfile` to build image based on the info in `vote/README.md`

```
FROM python:2.7-alpine
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 80
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80"]
```

- Build the image

```
docker build -t 81.81.81.11:1337/library/vote:v1 .
```

- Test running as container

```
docker run -it --rm -p 3773:80 81.81.81.11:1337/library/vote:v1
```

- Try to access application with browser

- Push image to registry

```
docker login http://81.81.81.11:1337
docker push 81.81.81.11:1337/library/vote:v1
```

## 2. Build a result application (node.js)

- Create `Dockerfile` to build image based on the info in `result/README.md`

```
FROM node:8.9-alpine
WORKDIR /app
COPY . .
RUN npm config set registry https://registry.npmjs.org
RUN npm install \
    && npm ls \
    && npm cache clean --force \
    && mv /app/node_modules /node_modules
ENV 80
EXPOSE 80
CMD ["node", "server.js"]
```

- Build the image

```
docker build -t 81.81.81.11:1337/library/result:v1 .
```

- Test running as container

```
docker run -it --rm -p 3773:80 81.81.81.11:1337/library/result:v1
```

- Try to access application with browser

- Push image to registry

```
docker login http://81.81.81.11:1337
docker push 81.81.81.11:1337/library/result:v1
```

## 3. Build a worker application (java)

- Create `Dockerfile` to build image based on the info in `result/README.md`

To reduce as much image size as possible, i will use `multi-stage Dockerfile`

```
FROM maven:3.5-jdk-8-alpine AS build
WORKDIR /code
COPY . .
RUN ["mvn", "package", "-DskipTests"]

FROM openjdk:8-jre-alpine AS run
WORKDIR /run
COPY --from=build /code/target/worker-jar-with-dependencies.jar worker.jar
CMD ["java", "-jar", "worker.jar"]
```

- Build the image

```
docker build -t 81.81.81.11:1337/library/worker:v1 .
```

- Test running as container

Make sure show `waiting for redis` logs

```
docker run -it --rm 81.81.81.11:1337/library/worker:v1
```

- Push image to registry

```
docker login http://81.81.81.11:1337
docker push 81.81.81.11:1337/library/worker:v1
```
