<h1 align=center> Create and Manage Cloud Resources: Challenge Lab Write-Up
</h1>

<h1 align=center>

![](https://cdn.qwiklabs.com/hrlQwoBPXpe4Zc9mivDlGHUQ8FWQWh8VPZbWXlX25W8%3D)

</h1>

Created a full lab write up for the  Create and Manage Cloud Resources: Challenge Lab (GSP313) for the ACE09 **Google Cloud JumpStart Program**.

Guide created by: [Joseph M](https://www.linkedin.com/in/ofcljm/)

<br></br>

## Task 1: Create a project Jumphost instance

### The first step is to create a Jumphost instance. In the GCP Console go to Navigation Menu > Compute Engine > VM Instance.

Add the following parameters for machine type, and Image type:

- The name of instance: **nucleus-jumphost**
- Region set as: **Default Region**
- Zone set as: **Default Zone**
- The machine type be: **f1-micro**
- Using the default image type: **Debian Linux**

![](/Images/VM%20instance.png)
<br></br>

## Task 2: Create a Kubernetes service cluster

### Create the kubernetes cluster within US EAST and a private VPC (nucleus-vpc):
```
gcloud container clusters create nucleus-backend \
          --num-nodes 1 \
          --network nucleus-vpc \
          --region us-east1
```
### get-creds from the nucleus backend:
```
gcloud container clusters get-credentials nucleus-backend \
          --region us-east1
```
### Deploy a "Hello World" web app server to the cluster:
```
kubectl create deployment hello-server \
          --image=gcr.io/google-samples/hello-app:2.0
```
### Expose the kubernetes web app to the internet via an HTTP loadbalancer on port 8081:
```
kubectl expose deployment hello-server \
          --type=LoadBalancer \
          --port 8081
```

## Task 3: Set up an HTTP load balancer

### Once your cluster has been created (can take 6+ mins...), paste this startup script in the command line all at once. This will tell the container to update, install nginx and start the nginx service at server start-up: 
```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```
### Next are going to need to create a web server from a compute engine template that will use the "startup.sh" script that we just created previously: 
```bash
gcloud compute instance-templates create web-server-template \
       --metadata-from-file startup-script=startup.sh \
       --network nucleus-vpc \
       --machine-type g1-small \
       --region us-east1
```
### Create the nginx pool ( the thread pool is performing the functions of the delivery service such as loadbalancing):
```bash
gcloud compute target-pools create nginx-pool
```
### Create a compute instance group for the web server with a size of 2:
```bash
gcloud compute instance-groups managed create web-server-group \
       --base-instance-name web-server \
       --size 2 \
       --template web-server-template \
       --region us-east1
```
### Creating a firewall rule to allow traffic (80/tcp)
```
gcloud compute firewall-rules create permit-tcp-rule-356 \
       --allow tcp:80 \
       --network nucleus-vpc
```
### Here we will create a basic http health check (you can reference this doc: https://cloud.google.com/load-balancing/docs/health-checks#gcloud_2) with the following:
```
gcloud compute http-health-checks create http-basic-check
```
### Creating a backend service and attach the managed instance group you have created previously:
```
gcloud compute instance-groups managed \
       set-named-ports web-server-group \
       --named-ports http:80 \
       --region us-east1
```
### Create the webserver backend with the health checks and to allow HTTP traffic:
```
gcloud compute backend-services create web-server-backend \
       --protocol HTTP \
       --http-health-checks http-basic-check \
       --global
```
### Once you create the backend, you will want to add the servergroup to it:
```
gcloud compute backend-services add-backend web-server-backend \
       --instance-group web-server-group \
       --instance-group-region us-east1 \
       --global
```
### Creating a URL map and target HTTP proxy to route requests to your URL map:
```
gcloud compute url-maps create web-server-map \
       --default-service web-server-backend

gcloud compute target-http-proxies create http-lb-proxy \
       --url-map web-server-map
```
### Creating forwarding rule:
```
gcloud compute forwarding-rules create permit-tcp-rule-261 \
     --global \
     --target-http-proxy http-lb-proxy \
     --ports 80
```
### List the forwarding rules that you have created to verify the creation of the HTTP Loadbalancer:
```
gcloud compute forwarding-rules list
```