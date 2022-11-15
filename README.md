# psc-producer-consumer
Private Service Connect uses endpoints and service attachments to let service consumers send traffic from the consumer's VPC network to services in the service producer's VPC network.

## Key concepts for service consumers
You can use Private Service Connect endpoints to consume services that are outside of your VPC network. Service consumers create Private Service Connect endpoints that connect to a target service.

### Endpoints

You use Private Service Connect endpoints to connect to a target service. Endpoints have an internal IP address in your VPC network and are based on the forwarding rule resource.

You send traffic to the endpoint, which forwards it to targets outside of your VPC network.

### Targets

Private Service Connect endpoints have a target, which is the service you want to connect to:

- An API bundle:
- All APIs: most Google APIs
- VPC-SC: APIs that VPC Service Controls supports
- A published service in another VPC network. This service can be managed by your own organization or a third party.
### Published service

To connect your endpoint to a service producer's service, you need the service attachment for the service. The service attachment URI has this format: projects/SERVICE_PROJECT/regions/REGION/serviceAttachments/SERVICE_NAME

## Key concepts for service producers
To make a service available to consumers, you create one or more dedicated subnets to use for network address translation (NAT) of consumer IP addresses. You then create a service attachment which refers to those subnets.

### Private Service Connect subnets

To expose a service, the service producer first creates one or more subnets with purpose Private Service Connect.

When a request is sent from a consumer VPC network, the consumer's source IP address is translated using source NAT (SNAT) to an IP address selected from one of the Private Service Connect subnets.

If you want to retain the consumer connection IP address information, see Viewing consumer connection information.

These subnets cannot be used for resources such as VM instances or forwarding rules. The subnets are used only to provide IP addresses for SNAT of incoming consumer connections.

The Private Service Connect subnet must contain at least one IP address for every 63 consumer VMs so that each consumer VM is allocated 1,024 source tuples for network address translation.

The minimum size for a Private Service Connect subnet is /24.

### Service attachments

Service producers expose their service through a service attachment.

To expose a service, a service producer creates a service attachment that refers to the service's load balancer forwarding rule.
To access a service, a service consumer creates an endpoint that refers to the service attachment.
### Connection preferences

When you create a service, you choose how to make it available. There are two options:

Automatically accept connections for all projects - any service consumer can configure an endpoint and connect to the service automatically.
Accept connections for selected projects - service consumers configure an endpoint to connect to the service and the service producer accepts or rejects the connection requests.

# Demo
## Create projects
psc-test for Producer and psc-test2 for Consumer.

Setup the env vars 
```
. ./setenv.sh
```
## Producer 
```
gcloud config list project
gcloud config set project $prodproject
echo $prodproject
gcloud compute networks create vpc-demo-producer --project=$prodproject --subnet-mode=custom
gcloud compute networks subnets create vpc-demo-us-west2 --project=$prodproject --range=10.0.2.0/24 --network=vpc-demo-producer --region=us-west2
gcloud compute routers create crnatprod --network vpc-demo-producer --region us-west2
gcloud compute routers nats create cloudnatprod --router=crnatprod --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges --enable-logging --region us-west2
```

Compute instance www-01
```
gcloud compute instances create www-01 \
    --zone=us-west2-a \
    --image-family=debian-9 \
    --image-project=debian-cloud \
    --subnet=vpc-demo-us-west2 --no-address \
    --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install tcpdump -y
apt-get install apache2 -y
a2ensite default-ssl
apt-get install iperf3 -y
a2enmod ssl
vm_hostname="$(curl -H "Metadata-Flavor:Google" \
http://169.254.169.254/computeMetadata/v1/instance/name)"
filter="{print \$NF}"
vm_zone="$(curl -H "Metadata-Flavor:Google" \
http://169.254.169.254/computeMetadata/v1/instance/zone \
| awk -F/ "${filter}")"
echo "Page on $vm_hostname in $vm_zone" | \
tee /var/www/html/index.html
systemctl restart apache2
iperf3 -s -p 5050'
```

Compute instance www-02
```
gcloud compute instances create www-02 \
    --zone=us-west2-a \
    --image-family=debian-9 \
    --image-project=debian-cloud \
    --subnet=vpc-demo-us-west2 --no-address \
    --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install tcpdump -y
apt-get install apache2 -y
a2ensite default-ssl
apt-get install iperf3 -y
a2enmod ssl
vm_hostname="$(curl -H "Metadata-Flavor:Google" \
http://169.254.169.254/computeMetadata/v1/instance/name)"
filter="{print \$NF}"
vm_zone="$(curl -H "Metadata-Flavor:Google" \
http://169.254.169.254/computeMetadata/v1/instance/zone \
| awk -F/ "${filter}")"
echo "Page on $vm_hostname in $vm_zone" | \
tee /var/www/html/index.html
systemctl restart apache2
iperf3 -s -p 5050'
```

Unmanaged instance group with www-01 & www-02
```
gcloud compute instance-groups unmanaged create vpc-demo-ig-www --zone=us-west2-a

gcloud compute instance-groups unmanaged add-instances vpc-demo-ig-www --zone=us-west2-a --instances=www-01,www-02

gcloud compute health-checks create http hc-http-80 --port=80
```
### Create TCP backend services, forwarding rule & firewall
```
gcloud compute backend-services create vpc-demo-www-be-tcp --load-balancing-scheme=internal --protocol=tcp --region=us-west2 --health-checks=hc-http-80
gcloud compute backend-services add-backend vpc-demo-www-be-tcp --region=us-west2 --instance-group=vpc-demo-ig-www --instance-group-zone=us-west2-a
gcloud compute forwarding-rules create vpc-demo-www-ilb-tcp --region=us-west2 --load-balancing-scheme=internal --network=vpc-demo-producer --subnet=vpc-demo-us-west2 --address=10.0.2.10 --ip-protocol=TCP --ports=all --backend-service=vpc-demo-www-be-tcp --backend-service-region=us-west2
gcloud compute firewall-rules create vpc-demo-health-checks --allow tcp:80,tcp:443 --network vpc-demo-producer --source-ranges 130.211.0.0/22,35.191.0.0/16 --enable-logging
gcloud compute firewall-rules create psclab-iap-prod --network vpc-demo-producer --allow tcp:22 --source-ranges=35.235.240.0/20 --enable-logging
```

### TCP NAT Subnet
```
gcloud compute networks subnets create vpc-demo-us-west2-psc-tcp --network=vpc-demo-producer --region=us-west2 --range=192.168.0.0/24 --purpose=private-service-connect
```
### Create TCP service attachment and firewall rules
```
gcloud compute service-attachments create vpc-demo-psc-west2-tcp --region=us-west2 --producer-forwarding-rule=vpc-demo-www-ilb-tcp --connection-preference=ACCEPT_AUTOMATIC --nat-subnets=vpc-demo-us-west2-psc-tcp
gcloud compute service-attachments describe vpc-demo-psc-west2-tcp --region us-west2
gcloud compute --project=$prodproject firewall-rules create vpc-demo-allowpsc-tcp --direction=INGRESS --priority=1000 --network=vpc-demo-producer --action=ALLOW --rules=all --source-ranges=192.168.0.0/24 --enable-logging
```
## Consumer 
Setup the env vars 
```
. ./setenv.sh
```
Setup project 
```
gcloud config list project
gcloud config set project $consumerproject
echo $consumerproject
```
### Setup VPC & Subnet
```
gcloud compute networks create vpc-demo-consumer --project=$consumerproject --subnet-mode=custom
gcloud compute networks subnets create consumer-subnet --project=$consumerproject  --range=10.0.60.0/24 --network=vpc-demo-consumer --region=us-west2
gcloud compute addresses create vpc-consumer-psc-tcp --region=us-west2 --subnet=consumer-subnet --addresses 10.0.60.100
gcloud compute firewall-rules create psclab-iap-consumer --network vpc-demo-consumer --allow tcp:22 --source-ranges=35.235.240.0/20 --enable-logging
gcloud compute --project=$consumerproject firewall-rules create vpc-consumer-psc --direction=EGRESS --priority=1000 --network=vpc-demo-consumer --action=ALLOW --rules=all --destination-ranges=10.0.60.0/24 --enable-logging
gcloud compute routers create crnatconsumer --network vpc-demo-consumer --region us-west2
gcloud compute routers nats create cloudnatconsumer --router=crnatconsumer --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges --enable-logging --region us-west2
```

Create a VM 
```
gcloud compute instances create test-instance-1 \
    --zone=us-west2-a \
    --image-family=debian-9 \
    --image-project=debian-cloud \
    --subnet=consumer-subnet --no-address \
    --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install iperf3 -y
apt-get install tcpdump -y'
```

TCP Service Attachment
```
gcloud compute forwarding-rules create vpc-consumer-psc-fr-tcp --region=us-west2 --network=vpc-demo-consumer --address=vpc-consumer-psc-tcp --target-service-attachment=projects/$prodproject/regions/us-west2/serviceAttachments/vpc-demo-psc-west2-tcp
```

Validate the static IP and forwarding rule
```
gcloud compute forwarding-rules describe vpc-consumer-psc-fr-tcp --region us-west2
```

TCP validation - SSH to Producer www-01
```
sudo tcpdump -i any net 192.168.0.0/16 -n
```
TCP validation - SSH to Producer www-02
```
sudo tcpdump -i any net 192.168.0.0/16 -n
```

TCP validation - SSH to Consumer 
```
sudo tcpdump -i any host 10.0.60.100 -n
```
SSH to Consumer - Access the static IP from the Forwarding rule
```
curl -v 10.0.60.100 
```

## Clean up Producer 
```
gcloud compute routers nats delete cloudnatprod --router=crnatprod --region=us-west2 --quiet

gcloud compute routers delete crnatprod --region=us-west2 --quiet

gcloud compute instances delete www-01 --zone=us-west2-a --quiet

gcloud compute instances delete www-02 --zone=us-west2-a --quiet

gcloud compute service-attachments delete vpc-demo-psc-west2-tcp --region=us-west2 --quiet

gcloud compute forwarding-rules delete vpc-demo-www-ilb-tcp --region=us-west2 --quiet

gcloud compute backend-services delete vpc-demo-www-be-tcp --region=us-west2 --quiet

gcloud compute instance-groups unmanaged delete vpc-demo-ig-www --zone=us-west2-a --quiet

gcloud compute health-checks delete hc-http-80 --quiet

gcloud compute firewall-rules delete vpc-demo-allowpsc-tcp --quiet

gcloud compute firewall-rules delete vpc-demo-health-checks --quiet

gcloud compute firewall-rules delete psclab-iap-prod --quiet

gcloud compute networks subnets delete vpc-demo-us-west2 --region=us-west2 --quiet

gcloud compute networks subnets delete vpc-demo-us-west2-psc-tcp --region=us-west2 --quiet

gcloud compute networks delete vpc-demo-producer --quiet
```

## Clean up Consumer
```
gcloud compute routers nats delete cloudnatconsumer --router=crnatconsumer --region=us-west2 --quiet

gcloud compute routers delete crnatconsumer --region=us-west2 --quiet

gcloud compute instances delete test-instance-1 --zone=us-west2-a --quiet

gcloud compute forwarding-rules delete vpc-consumer-psc-fr-tcp --region=us-west2 --quiet

gcloud compute addresses delete vpc-consumer-psc-tcp --region=us-west2 --quiet

gcloud compute firewall-rules delete psclab-iap-consumer --quiet

gcloud compute networks subnets delete consumer-subnet --region=us-west2 --quiet

gcloud compute firewall-rules delete vpc-consumer-psc --quiet

gcloud compute networks delete vpc-demo-consumer --quiet
```