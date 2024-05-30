## Push the project on github

`git init`

`git add *`

`git commit -m "first commit"`

`git branch -M main`

`git remote add origin https://github.com/Prospouille/DevOps.git`

`git push -u origin main`

## Build and run the tests
`mvn clean verify --file /Users/prosperplayoust/Desktop/EPF/4A/Devops/Docker/discover-docker/backend/simple-api-student-main/pom.xml`


### 2-1 What are testcontainers?

They are containers for development and testing use cases

### 2-2 Document your Github Actions configurations.

To configure the Github Actions, we need to create a workflow using a main.yml. 
This script will build and launch the tests on the api.

#### Workflow main.yml

name: CI devops 2024
on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      # Checkout your GitHub code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      # Set up JDK 17 using actions/setup-java@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      # Change directory to where the pom.xml file is located and build/test with Maven
      - name: Build and test with Maven
        working-directory: ./backend/simple-api-student-main
        run: mvn clean verify

## First steps into the CD World

To settle the continuous deployment, we want to add to the main.yml the creation and saving of the docker images. TO do that we add new jobs to connect to DockerHub, build the docker image by giving the path to the docker files and push them on DockerHub.

#### Workflow main.yml (suite)

#define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0


      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}


      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Docker/discover-docker/backend/simple-api-student-main
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKER_USERNAME}}/discover-docker-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with: 
            # relative path to the place where source code with Dockerfile is located
            context: ./Docker/discover-docker/database
            # Note: tags has to be all lower-case
            tags:  ${{secrets.DOCKER_USERNAME}}/database:latest
            push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./Docker/discover-docker/server
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKER_USERNAME}}/discover-docker-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}

## Setup Quality Gate

### Document your quality gate configuration.

We want to configure a quality gate to assure the quality of the code. 
To do that, we use sonarcloud.

We add this line in the previous main.yml

`mvn -B verify sonar:sonar -Dsonar.projectKey=devopsepf_docker -Dsonar.organization=devopsepf -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml`

This will connect to sonarcloud that will check our code. 

- Coverage : The part of the code that is checked by test (92.1%)
- Duplications : number of lines that is repeated (0%)
- Security Hotspots : Security breach ? (3)
- Accepted Issues (0)
- New Issues (2)

