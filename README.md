1. Create a service account
   Enable storage admin and compute admin permission
   storage@sql-i-370008.iam.gserviceaccount.com

2. Activate cloud shell and create VM and install apache2 

gcloud compute instances create webserver --project=sql-i-370008 --zone=us-west4-b --machine-type=e2-micro --network-interface=network-tier=PREMIUM,subnet=default --metadata=startup-script=\!\#bin/bash$'\n'apt\ update$'\n'sudo\ apt-get\ install\ apache2\ -y --no-restart-on-failure --maintenance-policy=TERMINATE --provisioning-model=SPOT --instance-termination-action=STOP --service-account=storage@sql-i-370008.iam.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --tags=http-server,https-server --create-disk=auto-delete=yes,boot=yes,device-name=webserver,image=projects/debian-cloud/global/images/debian-11-bullseye-v20221102,mode=rw,size=10,type=projects/sql-i-370008/zones/us-west4-b/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any


3. SSH into the VM and check the apache2 webserver installation status.
   
   sudo su
   apache2 -v
   

4. Create a bucket and upload the source code folder into the cloud shell.

   gsutil mb -c standard -l us-west4 gs://website-compute

   gsutil cp -v -r online-shop-website-template gs://website-source/

5. SSH INTO THE VM AND COPY SOURCE CODE FOLDER FROM BUCKET.

   cd /var/www/html/
   
   sudo rm index.html

   sudo gsutil -m cp -v -r gs://website-source/online-shop-website-template/* /var/www/html/

6. ACCESS THE WEBSITE PAGE BY CLICKING VM IP
    
   http://35.10.12.11

7. CREATE A CUSTOM IMAGE OUT OF THE RUNNING WEBSERVER

   STOP VM FIRST

   gcloud compute instances stop webserver --zone us-west4-b

   CREATE CUSTOM IMAGE OUT OF THE DISK

   gcloud compute images create webserver --project=sql-i-370008 --source-disk=webserver --source-disk-zone=us-west4-b --storage-location=us-west4


8. CREATE INSTANCE TEMPLATE


   gcloud compute instance-templates create webserver --project=sql-i-370008 --machine-type=e2-micro --network-interface=network=default,network-tier=PREMIUM --no-restart-on-failure --maintenance-policy=TERMINATE --provisioning-model=SPOT --instance-termination-action=STOP --service-account=9997358645-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --tags=http-server,https-server --create-disk=auto-delete=yes,boot=yes,device-name=webserver,image=projects/sql-i-370008/global/images/webserver,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any


9. CREATE HEALTH CHECK

   gcloud beta compute health-checks create http webserver --project=sql-i-370008 --port=80 --request-path=/ --proxy-header=NONE --no-enable-logging --check-interval=60 --timeout=30 --unhealthy-threshold=3 --healthy-threshold=3


10. CREATE FIREWALL RULES TO ALLOW HEALTH CONNECT TO CHECK VM
 
   gcloud compute firewall-rules create allow-health-check \
        --allow tcp:80 \
        --source-ranges 130.211.0.0/22,35.191.0.0/16 \
        --network default

11. CREATE INSTANCE GROUP

    gcloud beta compute instance-groups managed create webserver --project=sql-i-370008 --base-instance-name=webserver --size=1 --template=webserver --zone=us-west4-a --list-managed-instances-results=PAGELESS --health-check=webserver --initial-delay=30 
    && gcloud beta compute instance-groups managed set-autoscaling webserver --project=sql-i-370008 --zone=us-west4-a --cool-down-period=60 --max-num-replicas=8 --min-num-replicas=1 --mode=on --target-cpu-utilization=0.6


12. RESERVER STATIC IP GLOBAL

   gcloud compute addresses create webserver-loadbalancer --project=sql-i-370008 --global


13. CREATE LOAD HTTP loadbalancer > by console

-- name= webserver
-- frontend config=

-- new frontend ip and port = ecommerce-webserver
-- protocol= http 
-- port= 80
-- ip addresss= reserve static ip

-- backend config=

-- create backend= webserver
-- name= webserver
-- backend type= instance group
-- protocol= http
-- named port= http
-- timeout= 30 sec

-- Backends
-- edit backend

-- instance group= webserver
-- port numbers= 80, 443
-- balancing mode= utilization

-- Maximum backend utilization= 80
-- Maximum RPS= 30
-- Scope= per instance
-- Capacity= 100

Uncheck CDN


-- Health Check= webserver

Click on Create




