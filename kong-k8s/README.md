# KONG with oidc plugin for K8s setup

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