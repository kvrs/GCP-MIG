# Create VPC 
 gcloud compute networks create vpc-us-region --project=macro-entity-424704-j5 --subnet-mode=auto --mtu=1460 --bgp-routing-mode=regional 

# Create subnet for VPC
gcloud compute networks subnets create us-subnet1 --project=macro-entity-424704-j5 --range=0.0.0.0/0 --stack-type=IPV4_ONLY --network=vpc-us-region --region=us-central1

# create Firewall rule for VPC 
gcloud compute firewall-rules create vpc-us-region-allow-custom --project=macro-entity-424704-j5 --network=projects/macro-entity-424704-j5/global/networks/vpc-us-region --description=Allows\ connection\ from\ any\ source\ to\ any\ instance\ on\ the\ network\ using\ custom\ protocols. --direction=INGRESS --priority=65534 --source-ranges=192.168.194.0/24 --action=ALLOW --rules=all


# Create a instance template and Startup Script

gcloud compute instance-templates create apache-template \
    --machine-type n1-standard-1 \
    --network vpc-us-region  \
    --subnet us-subnet1\
    --image-family debian-11 \
    --image-project debian-cloud \
    --boot-disk-size 10GB \
    --boot-disk-type pd-standard \
    --metadata startup-script='#!/bin/bash
      sudo apt-get update
      sudo apt-get install -y apache2
      sudo systemctl start apache2
      sudo systemctl enable apache2' \
    --region us-central1

# Create a health check 
gcloud compute health-checks create tcp my-health-check   --port 80

# Create Managed instance group 
gcloud beta compute instance-groups managed create instance-group-1 \
    --project=macro-entity-424704-j5 \
    --base-instance-name=instance-group-1 \
    --template=projects/macro-entity-424704-j5/global/instanceTemplates/apache-template \
    --size=1 \
    --zone=us-central1-c \
    --default-action-on-vm-failure=repair \
    --health-check=projects/macro-entity-424704-j5/global/healthChecks/my-health-check \
    --initial-delay=300 \
    --no-force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=pageless \
&& \
gcloud beta compute instance-groups managed set-autoscaling instance-group-1 \
    --project=macro-entity-424704-j5 \
    --zone=us-central1-c\
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.6 \
    --cool-down-period=60


# Create a Python Script to Increase CPU Utilization 
# vi multi.py 
``` import multiprocessing
import time

def cpu_stress_test():
    while True:
        for i in range(10**6):
            is_prime = True
            for j in range(2, int(i**0.5) + 1):
                if i % j == 0:
                    is_prime = False
                    break

if __name__ == "__main__":
    cpu_count = multiprocessing.cpu_count()
    print(f"Starting stress test on {cpu_count} CPUs")

    processes = []
    for _ in range(cpu_count):
        p = multiprocessing.Process(target=cpu_stress_test)
        processes.append(p)
        p.start()

    # Keep the script running for a specific duration or indefinitely
    try:
        while True:
            time.sleep(10)
    except KeyboardInterrupt:
        print("Stopping stress test")
        for p in processes:
            p.terminate()
```
# apt install python3 
# python3 multi.py  



