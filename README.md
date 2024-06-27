# KONG with OIDC Plugin
Through this repo you can set up the KONG API Gateway with the OIDC plugin.

The OIDC plugin (which is available just in the KOng Entreprise edition) enables the connection with the 3rd party IAM system such as Keycloak. 


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

## Authentication & Authorization workflow using KONG & Keycloack 

The user application sends a request to Kong. However, the request is either not authenticated (or contains an invalid authentication). 

Kong responds to the client by indicating the lack of authentication. 

The application therefore needs to log in. Therefore, it sends a specific request for login to the Single Sign On (Keycloak), including the user's credentials and the specific client-id assigned to the application itself. 

If the credentials are valid, the SSO (Keycloak) issues to the application a token (and the related refresh token), with which to authenticate the requests to the Gateway API (Kong) 

The application then repeats the request adding the valid token as an authorization 

Behind the scenes, the gateway API will proceed to verify (through introspection) that the token in question corresponds to a session on the Single Sign On (Keycloak). 

The result of the introspection is returned to Kong, who will handle the application request accordingly 

If the outcome of introspection is positive, Kong will handle the request. Alternatively, we will be in step 2 (the request is refused) 

## Configuration on the Keycloak side  

We are using the Master realm 

We create a client (waves) with the following configuration: 

Client ID: waves 

Enabled: ON 

Client Protocol: openid-connect 

Access Type: confidential 

Standard Flow Enabled: ON 

Direct Access Grants Enabled: ON 

Root URL: http://localhost:8000 (http://{konghost}: {kongport}) 

Valid Redirect URIs: * 

Backchannel Logout Session Required: ON (setted on ON when we save the changes)  

After saving we can check the the Credentials tab and get the client secret (Secret: Z9pZR6ZpBfhUY54GFyf6kTFoUlECNxjf) 

For the users we used the admin user (username: admin & password: admin) 

## Configuration on the Kong Side  

We create a service with the following configuration: 

Name: waves 

Protocol: http 

Host: kong-konga-keycloak_waves-gateway_1 (name of the container of the upstream service that is running in the docker-compose network: in our case waves gateway) 

Port: 1111 (port of the container of the upstream service that is running in the docker-compose network: in our case the port of the waves gateway container) 

Path: /response (this is the endpoint that we are testing in the waves gateway) 

We create a route for the RESTful endpoint: 

We go to the routes tabs, and we create a route by clicking (+ADD ROUTE) 

where we just specify the value of “Paths: /response” 

We create a route for a websocket endpoint . We go to the routes tabs, and we create a route by clicking (+ADD ROUTE) and we fill the following configuration:  

Name: websocket 

Paths: /websocket 

Headers: “Upgrade:WebSocket” & “Connection:Upgrade” 

Url of to connect on : ws://0.0.0.0:8000/websocket/ws_notification/{user_id} 

We go to the plugins tabs after and we add the OIDC plugin by clicking (+ ADD PLUGIN) and choose in Other Oidc and we fill the following configuration: 

Bearer only: no 

Realm: master (Name of the used realm in keycloak, in our case we used the realm master) 

Client id: waves (the client id we specified in the keycloak) 

Client secret: HLPWwr3T3wVWYhyl5uSi6Ywzpo0jmpRq (the client secret specified in the keycloak) 

Discovery: http://172.18.0.1:8180/realms/master/.well-known/openid-configuration (you can get this discovrey URL by going to the Realm Settings tab in Keycloak and click on OpenID Endpoint Configuration in Endpoints) 

Scope: openid 

Response type: code 

Token endpoint auth method: client_secret_post 

Logout path: /logout 

Redirect after logout uri: /0.0.0.0 

Ssl verify : no 