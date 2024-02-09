# TP1

## Dockerfile (Question 1-1) 

Exemple simple d'un dockerfile permettant de monter une BDD Postgresql
```sh
FROM postgres:14.1-alpine

COPY init /docker-entrypoint-initdb.d

ENV POSTGRES_DB=postgres \
   POSTGRES_USER=postgres \
   POSTGRES_PASSWORD=postgres
```

- **FROM** permet de sélectionner la base du container, ici postgresql.
- **COPY** permet de copier le fichier présent sur la machine dans le répertoire destination du docker.
- **ENV** permet de déclarer des variables d'environnement sur le conteneur docker.


## Commandes Docker (Question 1-1) 

La commande suivante permet de build l'image docker en la nommant : 
```shell
docker build -t theo/postgres .
```

Ensuite on peut executer et lancer le conteneur docker postgres grâce à la commande suivante : 
```shell
docker run -p 5432:5432 --net=app-network --name postgres -v data:/var/lib/postgresql/data theo/postgres

-p pour exposer des ports du container sur notre machine 
--name nommer le container 
-v gestion des volumes
```

Pour lancer l'adminer on execute la commande suivante : 
```shell
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

Pour permettre la communication entre les deux container il est nécessaire de créer un network commun. Pour créer le network on utilise la commande suivante :
```shell
docker network create app-network
```


## Multistage build (Question 1-2) 

Voici le Dockerfile permettant de compiler et de run la partie backend de notre app:
```sh
# Stage 1 : Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY /simple-api-advanced/pom.xml .
COPY /simple-api-advanced/src ./src
RUN mvn package -DskipTests

# Stage 2 : Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

Le build en plusieurs étapes permet de réduire la taille de l'image Docker finale en séparant les dépendances de construction (Maven, code source, etc.) de l'environnement d'exécution (JRE, JAR d'application). La première étape crée les artefacts nécessaires, et la deuxième étape ne copie que les artefacts essentiels dans l'image finale. Cette façon de faire garantit que l'image finale ne contient que ce qui est nécessaire à l'exécution de l'application, ce qui permet d'obtenir une image Docker plus petite et plus efficace et plus légère.

## Docker-compose (Question 1-3 & 1-4) 

Main commands: 

Start the docker-compose
```shell
docker-compose up
```

Start the docker-compose in a detached mode 
```shell
docker-compose up -d
```

Stop and remove container from the docker-compose
```shell
docker-compose down
```   

Pause/Unpause containers in the docker-composer
```shell
docker-compose pause
docker-compose unpause
```   

Mon docker-compose :
```sh
version: '3.7'

services:
    backend:
        container_name: backend-advanced
        image: theoahga/my-backend:1.1
        networks:
          - app-network 
        depends_on:
          - database
        ports:
          - 8080:8080

    database:
        container_name: postgres
        image: theoahga/my-database:1.1
        networks:
          - app-network 
        ports:
          - 5432:5432
        volumes:
          - ./data:/app/

    httpd:
        container_name: apache
        image: theoahga/my-http:1.1
        networks:
          - app-network 
        ports:
          - 8040:80

networks:
  app-network:

```

## Publishing (Question 1-5) 

La publication d'une image se fait avec les commandes suivantes  
```shell
# Connecion au hub docker officiel
docker login

# Donner un tag à son image 
docker tag tp1-database theoahga/my-database:1.0

# Publier son image sur le hub
docker push theoahga/my-database:1.0
```

# TP2

## Testcontainers (Question 2-1)

Testcontainers est une bibliothèque Java qui fournit des instances légères et temporaires de bases de données courantes, de navigateurs web Selenium, ou de tout autre service pouvant s'exécuter dans un conteneur Docker pendant les tests. Elle est utilisée pour simplifier les tests d'intégration en gérant le cycle de vie des conteneurs nécessaires à vos tests.

## Github Action (Question 2-2)

Github Action permettant de tester le backend au merge sur la branch main : 
```yaml
name: CI devops 2023
on:
  push:
    branches: 
      - main 

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'adopt'
        cache: maven
    - name: Build with Maven
      run: cd backend/simple-api2/simple-api-advanced/ && mvn --batch-mode --update-snapshots verify

```

Github Action permettant de publier des images docker 

```yaml
name: publish_docker

on:
  workflow_run:
    workflows: [build_test]
    types:
      - completed
    branches: [main]


jobs: 
  publish_docker_hub:
    if : ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USER }} -p ${{ secrets.DOCKERHUB_PWD }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./backend/simple-api2/
          tags:  ${{secrets.DOCKERHUB_USER}}/my-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./postgres/
          tags:  ${{secrets.DOCKERHUB_USER}}/my-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}


      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http_server/
          tags:  ${{secrets.DOCKERHUB_USER}}/my-http:latest
          push: ${{ github.ref == 'refs/heads/main' }}

```

# TP3


## Ansible (Question 3-1)

Mon inventories/setup.yaml
```yaml
all:
 vars:
   ansible_user: "{{ USER }}"
   ansible_ssh_private_key_file: /id_rsa_takima
 children:
   prod:
     hosts: "{{ DNS }}"

```

Quelques commandes ansible:
```shell
# Pour ping 
ansible all -i inventories/setup.yml -m ping

# Checker les distribs
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"

# Remove apache
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

## Playbook (Question 3-2)

Un playbook avec des tâches directement dans le même fichier:
```yaml
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:

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

```

et maintenant en deux fichier :
```yaml

# playbook.yaml
- hosts: all
  gather_facts: false
  become: true

  roles:
   - docker



# roles/docker/task/main.yaml
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

```


Enfait on peut créer des rôles avec la commande suivante :
```sh
ansible-galaxy init roles/docker
```

Voici quelques rôles:
```yaml
# Création d'un network docker (create_network)
- name: create the common docker network 
  community.docker.docker_network:
    name: app-network

# Création du docker backend (launch_backend)
- name: Create the backend container
  community.docker.docker_container:
    name: backend-advanced
    image: "theoahga/my-backend:latest"
    networks:
      - name: app-network
    env:
        DBURL: "{{ DB_URL }}"
        DBUSER: "{{ DB_USER }}" 
        DBPWD: "{{ DB_PWD }}" 
```

## Ansible-vault

Un secret.yaml a été ajouté pour premettre de cacher certaine variables d'environnement grâce a ansible-vault (cf `ansible\secret` et les différents fichier .yaml dans lesquels les var sont utilisés)