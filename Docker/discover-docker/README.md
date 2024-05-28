#Database

###1-1 Document your database container essentials: commands and Dockerfile.

####Dockerfile

We want to import an image postgres to use for our database
`FROM postgres:14.1-alpine`

We create environment variables to configure our datase. It needs a name, a user and a password
`ENV POSTGRES_DB=db \`
   `POSTGRES_USER=usr \`
   `POSTGRES_PASSWORD=pwd`

We copy the sql files inside the container to get data inside our database.
`COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d`
`COPY 02-InsertData.sql /docker-entrypoint-initdb.d`

We want to build the database image.
`docker build -t pepsouille0/database .`

We pull the adminer that will be the web interface with the database
`docker pull adminer`

We create a network to allow the adminer and the database to comunicate
`docker network create database_prosp`

We create the contenair database in the network we have created and mount data to save the modification of the database. We give a password in case we don't want the password in the file.
`docker run -d --name database --network database_prosp -v ./data://var/lib/postgresql/data -e POSTGRES_PASSWORD=pwd pepsouille0/database`

We create the contenair adminer and put it in the network
`docker run -d --name adminer --network database_prosp -p 8081:8080 adminer`

#Backend API

We compile this Main to have a Main.class
`javac Main.java`  

####Dockerfile

We use a simple image to interpret java code
`FROM openjdk:17`

We want to launch this command at start of the container
`CMD ["java", "Main"]`

We build the image
`docker build -t pepsouille0/multistage_backend ./simpleapi`

We run the image giving a port to see
`docker run --name backend -it -p 8080:8080 pepsouille0/multistage_backend`

##Multistage build

###1-2 Why do we need a multistage build? And explain each step of this dockerfile.

The multistage use different images to build and run the app.

####Dockerfile

We use this image to build our app. It copies everything useful for the app to be build
# Build
`FROM maven:3.8.6-amazoncorretto-17 AS myapp-build`
`ENV MYAPP_HOME /opt/myapp`
`WORKDIR $MYAPP_HOME`
`COPY pom.xml .`
`COPY src ./src`
`RUN mvn package -DskipTests`

We use this image image to run the app. 
# Run
`FROM amazoncorretto:17`
`ENV MYAPP_HOME /opt/myapp`
`WORKDIR $MYAPP_HOME`
`COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar`

This is the entrypoint to our app
`ENTRYPOINT java -jar myapp.jar`

We build the image
`docker build -t pepsouille0/database_backend .`

We create the contenair and put it inside the network. We give a port to see its results
docker run --name backend_db -p 8080:8080 --network database_prosp pepsouille0/database_backend

#Http server

####Dockerfile

We use this image to get a web interface.
`FROM httpd:latest`

We copy this document to well configure the contenair that will be created in the future
`COPY httpd.conf /usr/local/apache2/conf/httpd.conf`

We copy the index.html to have a welcome page.
`COPY index.html /usr/local/apache2/htdocs/`

We build the image
`docker build -t pepsouille0/database_server .`

We create the contenair and give a port to see the results
`docker run --name server_bis -d -p 8082:8080  pepsouille0/database_server`