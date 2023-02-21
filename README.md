# Google Kubernetes Engine Workshop

## Code Repositories

[https://github.com/luigicfh/weather-app](https://github.com/luigicfh/weather-app)

[https://github.com/luigicfh/connectivity-app-client](https://github.com/luigicfh/connectivity-app-client)

[https://github.com/luigicfh/k8-workshop](https://github.com/luigicfh/k8-workshop)

---

## Setup

[Google Cloud setup checklist  |  Documentation](https://cloud.google.com/docs/enterprise/setup-checklist)

```bash
gcloud config set project <project-id>
```

```bash
export PROJECT=$(gcloud config get project)
```

---

## Container Registry

Clone repos and build images to store in Container Registry.

```bash
git clone https://github.com/luigicfh/weather-app.git
```

```bash
cd weather-app
```

```bash
gcloud builds submit --tag=gcr.io/$PROJECT/weather-app
```

```bash
git clone https://github.com/luigicfh/ping-app.git
```

```bash
cd ..
```

```bash
cd ping-app
```

```bash
gcloud builds submit --tag=gcr.io/$PROJECT/ping-app
```

---

## Network 1 (Kubernetes VPC)

Create and configure the VPC and subnet for the K8 cluster

VPC

```bash
gcloud compute networks create k8-workshop-vpc --project=$PROJECT --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
```

Subnet

```bash
gcloud compute networks subnets create k8-workshop-us --project=$PROJECT --range=10.10.0.0/24 --stack-type=IPV4_ONLY --network=k8-workshop-vpc --region=us-east1
```

Firewall HTTP Ingress

```bash
gcloud compute --project=$PROJECT firewall-rules create k8-workshop-allow-http --direction=INGRESS --priority=1000 --network=k8-workshop-vpc --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http
```

Firewall ICMP Egress

```bash
gcloud compute --project=$PROJECT firewall-rules create k8-workshop-allow-icmp --direction=EGRESS --priority=1000 --network=k8-workshop-vpc --action=ALLOW --rules=icmp --source-ranges=0.0.0.0/0
```

---

## Network 2 and GCE instance (acting as on-premise environment)

This step will be performed in the second part of the workshop in a different project or console, to showcase VPN connectivity between the two networks.

```bash
gcloud config set project <project-id>
```

```bash
export PROJECT=$(gcloud config get project)
```

VPC

```bash
gcloud compute networks create k8-workshop-vpc-2 --project=$PROJECT --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
```

Subnet Europe

```bash
gcloud compute networks subnets create k8-workshop-eu --project=$PROJECT --range=10.30.0.0/24 --stack-type=IPV4_ONLY --network=k8-workshop-vpc-2 --region=europe-west3
```

Subnet US

```bash
gcloud compute networks subnets create k8-workshop-us --project=$PROJECT --range=10.20.0.0/24 --stack-type=IPV4_ONLY --network=k8-workshop-vpc-2 --region=us-east1
```

Firewall ICMP Ingress

```bash
gcloud compute --project=$PROJECT firewall-rules create test-icmp --direction=INGRESS --priority=1000 --network=k8-workshop-vpc-2 --action=ALLOW --rules=icmp --source-ranges=0.0.0.0/0
```

Allow SSH on port 22 for internal test

```bash
gcloud compute --project=$PROJECT firewall-rules create test-ssh --direction=INGRESS --priority=1000 --network=k8-workshop-vpc-2 --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0
```

GCE (acting as on-premise server)

```bash
gcloud compute instances create test-instance --project=$PROJECT --zone=europe-west3-c --machine-type=e2-medium --network-interface=subnet=k8-workshop-eu,no-address --maintenance-policy=MIGRATE --provisioning-model=STANDARD --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/debian-cloud/global/images/debian-11-bullseye-v20230206,mode=rw,size=10,type=projects/$PROJECT/zones/us-central1-a/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

Ping the internal IP of the GCE instance.

```bash
ping -c 1 10.30.0.2
```

---

## GKE cluster creation and deployments

```bash
gcloud config set project <project-id>
```

```bash
export PROJECT=$(gcloud config get project)
```

```bash
export ZONE=us-east1-b
```

```bash
export CLUSTER=k8-workshop
```

```bash
export NETWORK=k8-workshop-vpc
```

```bash
export SUBNET=k8-workshop-us
```

```bash
gcloud config set compute/zone $ZONE
```

```bash
gcloud container clusters create $CLUSTER \
--machine-type e2-small \
--num-nodes 1 \
--network $NETWORK \
--subnetwork $SUBNET \
--autoprovisioning-network-tags http
```

```bash
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```

```bash
source <(kubectl completion bash)
```

```bash
git clone https://github.com/luigicfh/k8-workshop.git
```

Weather app deployment and service

```bash
cd k8-workshop/deployment
```

```bash
kubectl apply -f ./weather-app-front.yaml
```

```bash
cd ..
```

```bash
cd service
```

```bash
kubectl apply -f ./weather-app-service.yaml
```

Blue/Green Deployment

[Implementing deployment and testing strategies on GKE  |  Cloud Architecture Center  |  Google Cloud](https://cloud.google.com/architecture/implementing-deployment-and-testing-strategies-on-gke#perform_a_bluegreen_deployment)

```bash
cd..
cd deployment
```

```bash
kubectl apply -f ./weather-app-front-new.yaml
```

Modify weather-app-service.yaml to point to the new app selector. It should look like the example below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: weather-app-v2
  template:
    metadata:
      labels:
        app: weather-app-v2
        track: stable
    spec:
      containers:
        - name: weather-app-v2
          image: "gcr.io/k8-workshop-378401/weather-app"
          ports:
          - name: http1
            containerPort: 8080
          imagePullPolicy: Always
```

```bash
cd..
cd service
```

```bash
kubectl apply -f ./weather-app-service.yaml
```

Ping app deployment and service

```bash
cd k8-workshop/deployment
```

```bash
kubectl apply -f ./ping-app-front.yaml
```

```bash
cd ..
```

```bash
cd service
```

```bash
kubectl apply -f ./ping-app-service.yaml
```

```bash
gcloud container clusters delete $CLUSTER
```

## VPN Setup - Kubernetes VPC and GCE VPC

Execute the following commands on both projects.

Reserve a Public Static IP

```bash
gcloud compute addresses create k8-workshop-vpn-ip  \
    --region=us-east1
```

Peer Static IP

```bash
export PEER=<replace with peer static IP>
```

Kubernetes Static IP

```bash
export K8IP=<replace with k8-workshop static IP>
```

Key

```bash
export IKE_KEY=<replace with IKE key>
```

```bash
gcloud compute target-vpn-gateways create k8-workshop-vpn --project=$PROJECT --region=us-east1 --network=k8-workshop-vpc
```

```bash
gcloud compute forwarding-rules create k8-workshop-vpn-rule-esp --project=$PROJECT --region=us-east1 --address=$K8IP --ip-protocol=ESP --target-vpn-gateway=k8-workshop-vpn
```

```bash
gcloud compute forwarding-rules create k8-workshop-vpn-rule-udp500 --project=$PROJECT --region=us-east1 --address=$K8IP --ip-protocol=UDP --ports=500 --target-vpn-gateway=k8-workshop-vpn
```

```bash
gcloud compute forwarding-rules create k8-workshop-vpn-rule-udp4500 --project=$PROJECT --region=us-east1 --address=$K8IP --ip-protocol=UDP --ports=4500 --target-vpn-gateway=k8-workshop-vpn
```

```bash
gcloud compute vpn-tunnels create k8-workshop-vpn-tunnel --project=$PROJECT --region=us-east1 --peer-address=$PEER --shared-secret=$IKE_KEY --ike-version=2 --local-traffic-selector=0.0.0.0/0 --remote-traffic-selector=0.0.0.0/0 --target-vpn-gateway=k8-workshop-vpn
```

```bash
gcloud compute routes create k8-workshop-vpn-tunnel-route-1 --project=$PROJECT --network=k8-workshop-vpc --priority=1000 --destination-range=0.0.0.0/0 --next-hop-vpn-tunnel=k8-workshop-vpn-tunnel --next-hop-vpn-tunnel-region=us-east1
```
