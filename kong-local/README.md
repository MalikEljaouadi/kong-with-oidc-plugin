# KONG with oidc plugin for local setup:

Runing the provided docker-compose in this directory will provide:

- KONG instance (with OIDC plugin)
- KONGA Dashboard to overview KONG
- 1st Postgres instance used by KONG
- Keycloack instance
- 2nd Postgres instance used by Keycloak

The mentioned resources above, would set the complete infrastructure to achieve the communication betwenn KONG and Keycloak

## Local Set Up of KONG & Keycloak (Docker-compose version) 

We will be using the Dockerfile and docker-compose file pricised in the link: 

We build the kong image by running this command: docker-compose build kong 

Kong uses a database server (postgresql in our case). For this reason it is necessary to the database by launching the necessary migrations. First we start the kong-db service (docker-compose up –d kong-db) then we start the kong migrations (docker-compose run –rm kong migrations bootstrap) (PS: in case we are upgrading kong from previous versions, we may need to run migrations using the following command : docker-compose run –rm kong kong migra initialize tions up) kong 

At this point we can start kong: docker-compose up –d kong  

We verify that we have 2 services running: docker-compose ps 

We verify that the OIDC plugin is present on Kong: curl -s http://localhost:8001 .plugins.available_on_server.oidc . And we should get true as a result. 

We start Konga: docker-compose up –d konga jq 

We start the keycloak database service using: docker-compose up –d keycloak-db | 

We start the keycloak service using: docker-compose up –d keycloak 

We check that everything is standing with: docker-compose ps . And we should see all the containers running  