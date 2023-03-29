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

## KONG Set Up on K8s using Helm 

Kong using the Official Kong helm chart proposed by the Kong INC: I have tried to use the same image (based on the Kong official image + addition of the OpenID connect plugin written in lua) that I already used on the local deployment of Kong but I faced issues related to: 

Version compatibility between the Kong image and the Kong helm chart 

Version compatibility between the postgres db and the Kong image  

Version compatibility between the oidc plugin and the Kong image  

The unavailability of a dashboard that displays a global overview of the Kong API Gateway services, upstream, routes, the traffic, and the used plugins. In addition to the functional limitations that imposes such as: inability to connect the dashboard to many Kong instances. 

The complexity of the deployment since it is composed of 4 separate steps that should be applied in order, which are: deployment of postgres db instance, running a the boostrap migration job of the postgres db related to Kong. in addition to that I should verify the connectivity of kong with the db  

Kong using the Bitnami Kong helm chart + Konga pantsel helm chart: for all the reasons mentioned above I decided to move forward with the helm chart and image of Kong provided by Bitnami.  

First, I created a Kong image that supports the openid plugin based on the Bitnami kong image. To achieve this task: I have pulled the repo of Bitnami Kong (https://github.com/bitnami/bitnami-docker-kong) and overrided its Dockerfile (https://github.com/bitnami/bitnami-docker-kong/blob/master/2/debian-11/Dockerfile) With this new image (https://bitbucket.org/elyadata/apisix-helm-chart/src/kong-konga-integration/kong-k8s/Kong_bitnami/bitnami-kong-image/Dockerfile) And then pushed this image to my personal docker repository (https://hub.docker.com/layers/malekeljaouadi/bitnami-kong-with-oidc/latest/images/sha256-03cd3c6e1ea19b6d7cd4b377eddd3dc4b6543e35929bd362dec4a652ef016a87?context=repo) 

Then I created custom values files for my custom subchart (https://bitbucket.org/elyadata/apisix-helm-chart/src/kong-konga-integration/kong-k8s/Kong_bitnami/subchart_values/) Where : in subchart-values-1.yaml the kong services are of type ClusterIP (accessible just from inside the cluster). In subchart-values-2.yaml the kong services are of type LoadBalancer. Both subcharts are using the Kong Kubernetes Ingress controller and we should replace that with our Nginx Ingress Controller.  

Finally I deployed the konga helm chart (https://github.com/pantsel/konga/tree/master/charts/konga) And connect it to the deployed kong deployed instance just by providing the <admin-service-name>:<admin-service-port> on konga 

### Some useful Commands:  

helm repo add test-kong-release https://charts.bitnami.com/bitnami 

helm upgrade --install kong-rel-test test-kong-release/kong -n kong-konga -f subchart-values-1.yaml 

helm delete kong-rel-test -n kong-konga 

PS: connection in konga should be <the ip @ of the service of kong>:8001 (port of the http admin) 

https://github.com/bitnami/charts/tree/main/bitnami/kong