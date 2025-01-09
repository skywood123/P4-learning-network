# Containerized Jupyterhub application

## What this repository doing
1.  ```p4mn-jupyter folder``` contains dockerfile to build a jupyter notebook container that runs **mininet, bmv2(grpc), P4RuntimeShell**.
2.  ```jupyterhub-deploy-docker folder``` contains file to spin up jupyterhub.

## Prerequisites
---------------------------------
1.  Docker installed

## Setting up the lab
1.  Create the container image for p4mn.
```
    cd p4mn-jupyter
    sudo docker build -t p4mn .
```
3.  By default jupyterhub listen on port 8000. Modify the **docker-compose.yml** 100.100.2.2 with your desired IP address to listen on.
```
    cd jupyterhub-deploy-docker/basic-example/
    nano docker-compose.yml
       ports:
      - "100.100.2.2:8000:8000"
```
4.  Spin up jupyterhub
```
    sudo docker compose up
```
6.  Access your jupyterhub at http://<IP_Address>:8000
   
