# Docker Django Rest Framework PostgreSQL Vue JS


## Installation Docker Docker compose
 `sudo add-apt-repository ppa:wereturtle/ppa`
 
## Pipenv install
import fichier Pipfile [Dépôt]

`pipenv install`


## Initialisation du backend avec Dockerfile et DockerCompose
` touch Dockerfile && gedit Dockerfile `

```
# Pull base image
FROM python:3.6

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

RUN pip install pipenv
ADD Pipfile* /code/
WORKDIR /code
RUN pipenv install --system --ignore-pipfile

ADD . /code/
```

` touch docker-compose.yml && gedit docker-compose.yml `

```
version: '3'

services:
  db:
    image: postgres
  backend:
    build: .
    command: python3 manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    depends_on:
      - db
```      

* lancement de la création du projet

`sudo docker-compose run backend django-admin.py startproject backend .`

* Le projet Django backend est créé

`ls -la`

* Parce qu'il a été créé avec docker-compose , changer les droits pour permettre à l'utilisateur d'y accéder .

`sudo chown -R $USER:$USER .`

* Réorganiser le projet

```
mkdir back && mv Dockerfile manage.py Pipfil* backend/ back
mv back backend
```

* Définition de la base de données dans Django

backend/backend/settings.py 

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
```

## Lancement d'un script de migration et runserver , modification du docker-compose

* Mise en place d'un script de lancement des migrations et du serveur lors du lancement du container

/backend  même niveau que le projet Django

```
mkdir scripts
cd scripts
touch start.sh && gedit start.sh
```

start.sh

```
#!/bin/bash

cd backend
python3 manage.py collectstatic --no-input
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:8000

```

rendre le script exécutable

``` 
sudo chmod +x backend/scripts/start.sh 
```


* Gestions des fichiers statiques

backend/backend/settings.py 
ajout à la fin 

```
STATIC_ROOT = 'static'
```

Note: on peut rajouter un fichier backend/static/.gitignore contenant 

```
*
!.gitignore
```

* Modification du Dockerfile (ajout du script)

```
# Pull base image
FROM python:3.6

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

#WORKDIR /app

RUN pip install pipenv
ADD Pipfile* /code/
WORKDIR /code
RUN pipenv install --system --ignore-pipfile
COPY scripts/start.sh /
ADD . /code/

```

* Modification du docker-compose (ajout du script et du nom des containers)

docker-compose.yml

```
version: '3'

services:
  db:
    container_name: db		
    image: postgres
  backend:
    container_name: backend
    build: ./backend
    command: /start.sh
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    depends_on:
      - db


```

## Lancement des containers:

```
docker-compose up --build
```


## Tester l'application Django et lancer des test avec Docker

backend/backend.tests.py

```
from django.contrib.auth.models import User
from django.test import TestCase

class TestDatabase(TestCase):
    def test_create_user(self):
        user = User.objects.create_user(
            username='user',
            email='user@foo.com',
            password='pass'
        )
        user.save()
        user_count = User.objects.all().count()
        self.assertEqual(user_count, 1)
```

pour lancer ce test 

```
docker-compose run backend python3 backend/manage.py test backend
```

## Mise en place du container frontend : VueJS

```
mkdir frontend

docker run --rm -it -v $PWD/frontend:/code node:12.2.0-alpine /bin/sh
```

Nous sommes dans le container

```
# cd code
# npm i -g vue @vue/cli
# vue create .

```

Après vue create  choisir les données suivantes

   
```
    Create the project in the current folder? Y
    Please pick a preset: Manually select features
    Check the features needed for your project: (select all except for TypeScript)
        Babel
        PWA
        Router
        Vuex
        CSS Pre-processors
        Linter /Formatter
        Unit Testing
        E2E Testing
    User history mode for router? Y
    Pick a CSS pre-processor Sass/SCSS
    Pick a linter / formatter config: ESLint + Airbnb config
    Pick additional lint features: Lint on save
    Pick a unit testing solution: Jest
    Pick a E2E testing solution: Nightwatch (Selenium-based)
    Where do you prefer placing config for Babel, PostCSS, ESLint, etc.? In package.json
    Save this as a preset for future projects? N
    Pick the package manager to use whne installing dependencies: Use NPM
    
    
```    
   


* changer les droits 

```
cd frontend
sudo chown -R $USER:$USER .
```


* attention ajout dans le .gitignore 

```
node_modules/*
```


## Création du script de lancement du frontend

```
cd frontend && touch start.sh && gedit start.sh 


#!/bin/bash

# https://docs.npmjs.com/cli/cache
npm cache verify

# install project dependencies
npm install

# run the development server
npm run serve
```

## Ajout du Dockerfile frontend

```
cd frontend
touch Dokerfile && gedit Dockerfile

FROM node:12.2.0-alpine

# make the 'app' folder the current working directory
WORKDIR /app/


# copy project files and folders to the current working directory (i.e. 'app' folder)
COPY . .

# expose port 8080 to the host
EXPOSE 8080

CMD ["sh", "start.sh"]
```

## Modification du docker-compose

```
version: '3'

services:
  db:
    container_name: db
    image: postgres
    volumes:
      - ./db-data:/var/lib/postgresql/10/docker_db
    networks:
      - main

  backend:
    container_name: backend
    build: ./backend
    command: /start.sh
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    networks:
      - main  
    depends_on:
      - db

  frontend:
    build:
      context: ./frontend
    volumes:
      - ./frontend:/app/frontend:ro
      - '/app/node_modules'
    ports:
      - "8080:8080"
    networks:
      - main
    depends_on:
      - backend
      - db
    environment:
      - NODE_ENV=development


networks:
  main:
    driver: bridge
    
```


## Création d'un super-utilisateur via Docker

`docker exec -it backend python3 backend/manage.py createsuperuser`

## Création d'une app Django via Docker

A la racine du projet

`docker exec -it backend python3 backend/manage.py startapp myapp`

`sudo chown -R $USER:$USER .`

`mv myapp backend/`
`
