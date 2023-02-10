1. Create a GCP project inside our GCP console. We have a project named ‘ocp-project’ created already, and this will be used to host the OCP 4.10 cluster.
   1. Go to **IAM & Admin** > **Create a Project**.
   2. Enter the project name & organization and click on *Create*.

2. Create a service account under IAM:
   1. Go to **IAM & admin** > **Service Accounts**
   2.	Click on **Create Service Account** and enter the relevant details:
         1.	Service Account name: *ocp-serviceaccount* (you can use any service account name of your choice)
         2.	Grant access to the necessary roles per the documentation. I have assigned it the **owner** role since I will be using this service account myself. It is however not recommended to grant the ‘owner’ role to the service account as it grants administrative privileges (full access) across your GCP account, and is not a security best practise. Do refer the necessary roles that the service account requires access to.
         3.	Click **Done**.
         4.	Once your service account is created, we need its json key to be used for authentication & authorization of gcp objects creation:
             1. Click on your service account name from the service accounts list.
             2. Go to the **keys** tab.
             3. Click on **Add key** > **Create New Key** and follow the instructions to create and download the json key.


3. Create a Network from the GCP UI Console:
   1.	Go to **VPC Networks** > **Create VPC Networks**
   2. Put in the appropriate network name. *ocp-network* is used in this exercise. (NOTE : *if you want to use different names for the network & subnetworks, and different IP subnet ranges; please ensure you also edit the sample-install-config.yaml file accordingly so that it contains the correct network & subnetwork names as well as the correct IP subnet ranges.*)
   3. Add 2 new subnets within this network:
         1. For the Master Nodes & Bootstrap node:
             1. Name: master-subnet
             2. Region: asia-northeast1 (*you can choose any region of your choice. In this exercise , asia-northeast1 is the nearest to my location and provides all necessary resource limits*)
             3. Subnet range: 10.1.10.0/24
             4.	Private Google Access: On
             5.	Flow Logs: Off
         2.	For the worker nodes:
             1.	Name: worker-subnet
             2.	Region: asia-northeast1 (*you can choose any region of your choice. In this exercise , asia-northeast1 is the nearest to my location and provides all necessary resource limits*)
             3.	Subnet Range: 10.1.20.0/24
             4.	Private Google Access: On
             5.	Flow Logs: Off
   4. For the Firewall rules, select all the 'allow' specific rules. Especially the rule that allows all communication between all the instances of the network. The rule name should be in the format of `<yournetworkname>-allow-custom`.
   5. Click **Create**.

4. Create a Firewall Rule to allow all traffic communication to and from the bastion host.
   1. Go to **'VPC Network'** > **'Firewall'**
   2. Enter the relevant details:
         1. Name: *allow-all-bastion*
         2. Logs: Off
         3. Network: ocp-network
         4. Priority: 100
         5. Direction of Traffic: Ingress
         6. Action on Match: Allow
         7. Targets: Specified Target Tags
            1. Target Tags: *'bastion'* (This is the network tag that will be assigned to the bastion host when it will be created).
         8. Source Filter: IPv4 Ranges
         9. Source IPv4 Ranges: 0.0.0.0/0
         10. Protocols & Ports: Allow All
         11. Click **Create**.

5. Create a ‘Cloud Router’ from the GCP console.
   1. Go to **‘Hybrid Connectivity’** > **‘Cloud Routers’**. (If *'Hybrid Connectivity'* option is not there in your left navigation menu, click on 'MORE PRODUCTS' option from the left navigation menu >  scroll down and you should find *'Hybrid Connectivity'* option. Click on the 'pin' icon so that it can be pinned to your pinned services list)
   2. Click on **‘Create Router’**.
   3.	Enter the relevant details:
         1. Name: ocp-router
         2. Network: ocp-network
         3. Region: asia-northeast1
         4. Select *“Advertise all subnets visible to the Cloud Router (Default)”*.
         5. Click **Create**..

6. Create 2 Cloud NAT components connected to the router we created above, for both of our subnets created earlier.
   1. Go to **‘Network Services’** > **‘Cloud NAT’**. (If *'Network Services'* option is not there in your left navigation menu, click on 'MORE PRODUCTS' option from the left navigation menu >  scroll down and you should find *'Network Services'* option. Click on the 'pin' icon so that it can be pinned to your pinned services list)
   2. Enter the relevant details:
         1. Gateway Name: ocp-nat-master-gw
         2. Network: ocp-network
         3. Region: asia-northeast1
         4. Cloud Router: ocp-router (this is the name of the cloud router we created in the previous step)
         5. NAT Mapping  
             1. Source: Custom
             2. Subnets: master-subnet
             3. NAT IP Addresses: Automatic
   3. Click **Create** .
   4. Repeat steps i, ii & iii again for the worker-subnet, and ensure you change the Gateway Name to ocp-nat-worker-gw for example. And ensure both Cloud NATs are connected to the same Cloud Router.

7. Create a private DNS zone from the GCP UI:
   1. Go to **‘Network Services’** > **‘Cloud DNS’**.
   2. Click on **‘Create Zone’** and enter the relevant details:
         1. Zone type: Private
         2. Zone name: ocp-private-zone (you can use any name of your choice)
         3. DNS name: vismishr.openshift.com (The DNS name consists of a combination of the cluster name & base domain i.e <cluster_name>.<base_domain> . In this example ‘vismishr’ is the name of my cluster and ‘openshift.com’ is the base domain that I will be using. Ensure you enter the values correctly as the cluster name & base domain you specify here will be the same that will have to be used in the install-config.yaml file later.
         4. Network: ocp-network (this needs to be the network we created earlier within which our master-subnet & worker-subnet reside)
         5.	Click **‘Create’**.

8. Create a bastion host in the master-subnet of the ocp-network we created earlier. Bastion host is basically a normal VM instance that can be used to log into and run the necessary commands from. Ensure it has an external IP assigned to it as we will be ssh’ing into it and run all the necessary commands from there.
   1. Go to **Compute Engine** > **‘VM Instances’** > **‘Create Instance’**.
   2. Put in the relevant details: Instance name, type (e2-medium is used for my demo), region, zone.
   3.	For the boot disk, I have used Ubuntu 18.04 OS with 50 GB disk size.
   4.	Identity & API Access:
         1. Service account: Compute Engine Default Service Account
         2. Access Scopes: Allow full access to all Cloud APIs (**This option is necessary as only then will you be able to execute the gcloud commands**)
   5. Firewall: Tick both checkboxes for allowing HTTP & HTTPS traffic.
   6.	For the Networking:
         1. Network Tags: *'bastion'* (Ensure you set this tag, so that the firewall rule we created in step 4 above is applicable to this bastion host)
         2. Assign a hostname of your choice. 
         3. Connect the network interface to the ocp-network, and subnet of master-subnet. You can set both Primary Internal IP & External IP to ‘Ephemeral’.
         4.	Keep IP forwarding enabled.
         5. Network Interface Card: VirtIO
   7.	Under **‘Security’** > **‘Manage Access’** -  ensure you add a ssh public key that allows you to ssh into the bastion host from your local machine.
   8.	Click **‘Create’**.

### OpenShift Installation 

9. SSH into the bastion host from your local machine, and switch to the root user. The *username* value is usually the local username that you use on your local machine. 
        
       ssh -i ~/.ssh/id_rsa <username>@<Public IP of bastion host>
       sudo su -
10. Copy over the downloaded json key (from step 2 above) to your bastion host, and save it by the name of `service-account-key.json`, as this is the service account key filename referenced in some of our commands later in this exercise. You can choose any name, just ensure you edit the commands accordingly.

11. Download some important packages:

        sudo apt update
        sudo apt install wget git -y
    
12. Download the necessary cli binaries 
    1. openshift-install cli binary
    2. oc cli binary
    3.	kubectl cli binary
    4.	RedHat Coreos image
    5.	Pull Secret
    6.	gcloud cli binary (Can be downloaded from [here](https://cloud.google.com/sdk/docs/install))
    7.	jq binary `sudo apt install jq -y`

13. Once downloaded and extracted, copy the openshift-install, oc & kubectl binary into /usr/local/bin/ directory (Or whatever the $PATH you have configured on your bastion host).
        
        tar xvf openshift-install-linux.tar.gz
        tar xvf openshift-client-linux.tar.gz
        mv oc kubectl openshift-install /usr/local/bin/


14. Generate a new ssh key pair on your bastion host keeping all default options. The public key from this key pair will be inserted in your install-config.yaml file. Your cluster nodes will be injected with this ssh key and you will be able to ssh into them later for any kind of monitoring & troubleshooting.
             
        ssh-keygen

        
15. Edit the  install-config.yaml as per your environment. The important changes are:
    1. Base Domain value (Needs to be the same as specified in your private DNS zone)
    2. Cluster name (Needs to be the same as specified in your private DNS zone)
    3. Pull Secret (To be taken from your RedHat account portal)
    4. SSH key (The one you created in step 14 above. The public key is present at `~/.ssh/id_rsa.pub` on most Linux machines)
    5. GCP platform specific parameters like project ID, region will have to specified per your choice. 

16. Export the GCP application credentials. This is the service account key that will be used for creating the remaining cluster components.

        export GOOGLE_APPLICATION_CREDENTIALS=<full path to your service account key json file>
        
27. Create the manifest files for your OpenShift cluster.

        openshift-install create manifests --dir install_dir/
    
18. The manifests need some changes to be made as follows:
    1. Open the file install_dir/manifests/cluster-ingress-default-ingresscontroller.yaml 

           vi install_dir/manifests/cluster-ingress-default-ingresscontroller.yaml
           
    2. Under spec.endpointPublishingStrategy :
       1. Remove the ‘loadbalancer’ parameter completely so that only the ‘type’ section remains.
       2. For the ‘type’ parameter, change the value to ‘HostNetwork’.
       3. Add the parameter of ‘replicas: 2’ below the ‘type’.
       4. Your resulting file should have the section look something like:
                  
              spec:
                endpointPublishingStrategy:
                  type: HostNetwork
                  replicas: 2
                  
       5. Save the file.
     
       Refer the sample-cluster-ingress-default-ingresscontroller.yaml file to compare and see how the resulting file should look like.
       
    3. Remove the manifest files for the worker & master machines, as we will be creating the master & worker nodes using the Deployment Manager templates.
              
           rm -f install_dir/openshift/99_openshift-cluster-api_master-machines-*
           rm -f install_dir/openshift/99_openshift-cluster-api_worker-machineset-*

19. Now let’s create the ignition config files.

        openshift-install create ignition-configs --dir install_dir/
          
20. Set the environment variables for your environment. Please set the values as per your needs.

        export BASE_DOMAIN=openshift.com
        export BASE_DOMAIN_ZONE_NAME=ocp-private-zone
        export NETWORK=ocp-network
        export MASTER_SUBNET=master-subnet
        export WORKER_SUBNET=worker-subnet
        export NETWORK_CIDR='10.1.0.0/16'
        export MASTER_SUBNET_CIDR='10.1.10.0/24'
        export WORKER_SUBNET_CIDR='10.1.20.0/24'
        export KUBECONFIG=/root/install_dir/auth/kubeconfig 
        export CLUSTER_NAME=`jq -r .clusterName /root/install_dir/metadata.json`
        export INFRA_ID=`jq -r .infraID /root/install_dir/metadata.json`
        export PROJECT_NAME=`jq -r .gcp.projectID /root/install_dir/metadata.json`
        export REGION=`jq -r .gcp.region /root/install_dir/metadata.json`
        export CLUSTER_NETWORK=(`gcloud compute networks describe ${NETWORK} --format json | jq -r .selfLink`)
        export CONTROL_SUBNET=(`gcloud compute networks subnets describe ${MASTER_SUBNET} --region=${REGION} --format json | jq -r .selfLink`)
        export COMPUTE_SUBNET=(`gcloud compute networks subnets describe ${WORKER_SUBNET} --region=${REGION} --format json | jq -r .selfLink`)
        export ZONE_0=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[0] | cut -d "/" -f9`)
        export ZONE_1=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[1] | cut -d "/" -f9`)
        export ZONE_2=(`gcloud compute regions describe ${REGION} --format=json | jq -r .zones[2] | cut -d "/" -f9`)
      
    These environment variables will be required for the upcoming commands as they call these variables in them, so be sure to set these.


21. Now we create the internal loadbalancer component that will be used for api related communication by the cluster nodes. The following commands will create the load balancer, its corresponding health check component, as well as the backend empty instance groups into which the master nodes will be put into at the time of Master nodes creation in a later step.
    1. Create the `.yaml` file
       
           cat <<EOF >02_infra.yaml
           imports:
           - path: 02_lb_int.py 
           resources: 
           - name: cluster-lb-int
             type: 02_lb_int.py
             properties:
               cluster_network: '${CLUSTER_NETWORK}'
               control_subnet: '${CONTROL_SUBNET}' 
               infra_id: '${INFRA_ID}'
               region: '${REGION}'
               zones: 
               - '${ZONE_0}'
               - '${ZONE_1}'
               - '${ZONE_2}'
           EOF
    2. Create the corresponding GCP object:
    
           gcloud deployment-manager deployments create ${INFRA_ID}-infra --config 02_infra.yaml
           
           
22. We now need to get the Cluster IP. This is basically the loadbalancer IP that we created in the previous step. This IP is used as the Host IP addresses for the DNS record sets that will be put into our private DNS zone. 

        export CLUSTER_IP=(`gcloud compute addresses describe ${INFRA_ID}-cluster-ip --region=${REGION} --format json | jq -r .address`)
        
23. We now create the required record sets for api communication among the cluster nodes.

        if [ -f transaction.yaml ]; then rm transaction.yaml; fi
        gcloud dns record-sets transaction start --zone ocp-private-zone
        gcloud dns record-sets transaction add ${CLUSTER_IP} --name api.${CLUSTER_NAME}.${BASE_DOMAIN}. --ttl 60 --type A --zone ocp-private-zone
        gcloud dns record-sets transaction add ${CLUSTER_IP} --name api-int.${CLUSTER_NAME}.${BASE_DOMAIN}. --ttl 60 --type A --zone ocp-private-zone
        gcloud dns record-sets transaction execute --zone ocp-private-zone
       
24. Let’s export the service account emails as these will be called inside the deployment manager templates of our master & worker nodes. In our demo, we will be using the same service account we created earlier:

        export MASTER_SERVICE_ACCOUNT=(`gcloud iam service-accounts list --filter "email~^ocp-serviceaccount@${PROJECT_NAME}." --format json | jq -r '.[0].email'`)
        export WORKER_SERVICE_ACCOUNT=(`gcloud iam service-accounts list --filter "email~^ocp-serviceaccount@${PROJECT_NAME}." --format json | jq -r '.[0].email'`)
     
     If the above commands do not set the correct service account email ENV variable, you can try running `gcloud iam service-accounts list`, copy the email address for your service account from the output's email column and manually set it as the environment variable for both `MASTER_SERVICE_ACCOUNT` & `WORKER_SERVICE_ACCOUNT`.
     
25. Now let’s create 2 google cloud buckets. One will be for storing the bootstrap.ign file for the bootstrap node, and the other will be the one to store the RedHat Coreos image that the cluster nodes will pull to boot up from.
    1. Bucket to store bootstrap.ign file.
    
           gsutil mb gs://${INFRA_ID}-bootstrap-ignition
           gsutil cp install_dir/bootstrap.ign gs://${INFRA_ID}-bootstrap-ignition/
           
    2. Bucket to store RedHat Coreos file. Please take note that in the below commands, the filename of the rhcos image is `rhcos-4.10.3-x86_64-gcp.x86_64.tar.gz`. In your case the name might be different, so please replace the value accordingly. Also, the bucket names need to be unique globally, so in your case, you might be unable to use the bucket name as *'rhcosbucket'*, instead you can use any other arbitrary name of your choice, and just ensure you replace the the name correctly in any of the commands referencing the bucket name.

           gsutil mb gs://rhcosbucket/
           gsutil cp rhcos-4.10.3-x86_64-gcp.x86_64.tar.gz gs://rhcosbucket/
           export IMAGE_SOURCE=gs://rhcosbucket/rhcos-4.10.3-x86_64-gcp.x86_64.tar.gz
           gcloud compute images create "${INFRA_ID}-rhcos-image" --source-uri="${IMAGE_SOURCE}"
           
26. Let’s set the CLUSTER_IMAGE env to be called later by the node creation commands.

        export CLUSTER_IMAGE=(`gcloud compute images describe ${INFRA_ID}-rhcos-image --format json | jq -r .selfLink`)

27. Let’s set the BOOTSTRAP_IGN env to be called in the next step of bootstrap node creation.
       
        export BOOTSTRAP_IGN=`gsutil signurl -d 1h service-account-key.json gs://${INFRA_ID}-bootstrap-ignition/bootstrap.ign | grep "^gs:" | awk '{print $5}'`

28. Now we create the bootstrap node itself using the relevant Deployment Manager template. The below commands will create the bootstrap node, a public IP for it, and an empty instance group for the bootstrap node:
    1. Create the `.yaml` config file for the bootstrap node deployment:
    
           cat <<EOF >04_bootstrap.yaml
           imports:
           - path: 04_bootstrap.py

           resources:
           - name: cluster-bootstrap
             type: 04_bootstrap.py
             properties:
               infra_id: '${INFRA_ID}' 
               region: '${REGION}' 
               zone: '${ZONE_0}' 

               cluster_network: '${CLUSTER_NETWORK}' 
               control_subnet: '${CONTROL_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-2' 
               root_volume_size: '50' 

               bootstrap_ign: '${BOOTSTRAP_IGN}' 
           EOF
    2. Run the command to create the bootstrap node.

           gcloud deployment-manager deployments create ${INFRA_ID}-bootstrap --config 04_bootstrap.yaml
           
        
29. Now we need to manually add the bootstrap node to the new empty instance group and add it as part of the internal load balancer we created earlier. This is mandatory as the initial temporary bootstrap cluster is hosted on the bootstrap node.

        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-bootstrap-instance-group --zone=${ZONE_0} --instances=${INFRA_ID}-bootstrap
        gcloud compute backend-services add-backend ${INFRA_ID}-api-internal-backend-service --region=${REGION} --instance-group=${INFRA_ID}-bootstrap-instance-group --instance-group-zone=${ZONE_0}

30. We now proceed with the master nodes creation. 
    1. Set the env variable for the master ignition config file.

           export MASTER_IGNITION=`cat install_dir/master.ign`

    2. Create the `.yaml` config file for the master nodes deployment:

           cat <<EOF >05_control_plane.yaml
           imports:
           - path: 05_control_plane.py

           resources:
           - name: cluster-control-plane
             type: 05_control_plane.py
             properties:
               infra_id: '${INFRA_ID}' 
               zones: 
               - '${ZONE_0}'
               - '${ZONE_1}'
               - '${ZONE_2}'

               control_subnet: '${CONTROL_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '100'
               service_account_email: '${MASTER_SERVICE_ACCOUNT}' 

               ignition: '${MASTER_IGNITION}' 
           EOF
      
      3. Run the command to create the master nodes:
    
        gcloud deployment-manager deployments create ${INFRA_ID}-control-plane --config 05_control_plane.yaml
        
31. Once the master nodes have been deployed, we also need to add them to their respective instance groups that were created earlier in the load balancer creation step:

        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_0}-instance-group --zone=${ZONE_0} --instances=${INFRA_ID}-master-0
        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_1}-instance-group --zone=${ZONE_1} --instances=${INFRA_ID}-master-1
        gcloud compute instance-groups unmanaged add-instances ${INFRA_ID}-master-${ZONE_2}-instance-group --zone=${ZONE_2} --instances=${INFRA_ID}-master-2
        
32. At this point we must now wait for the bootstrap process to complete. You can now monitor the bootstrap process:
    1. ssh into your bootstrap node from your bastion host (`ssh -i ~/.ssh/id_rsa core@<ip of your bootstrap node>`) and run `journalctl -b -f -u release-image.service -u bootkube.service` . Upon bootstrap process completion, the output of this command should stop at a message that looks something like `systemd[1]: bootkube.service: Succeeded`. 


33. Once our bootstrap process completes, we can remove the bootstrap components:

        gcloud compute backend-services remove-backend ${INFRA_ID}-api-internal-backend-service --region=${REGION} --instance-group=${INFRA_ID}-bootstrap-instance-group --instance-group-zone=${ZONE_0}
        gsutil rm gs://${INFRA_ID}-bootstrap-ignition/bootstrap.ign
        gsutil rb gs://${INFRA_ID}-bootstrap-ignition
        gcloud deployment-manager deployments delete ${INFRA_ID}-bootstrap

34. Now we are good to create the worker nodes. Note that if we run an `oc get co` command from the bastion host at this time, we will still see a few operators as *'unavailable'*, typically the ingress, console, authentication, and a few other cluster operators. This is because they depend on some components which need to come up on the worker nodes. For example, the ingress cluster operator will deploy the router pods on the worker nodes by default, and only then will a route to the console be created, and eventually the authentication operator would also reach completion.
    1. Set the env variable for the worker ignition config file.

           export WORKER_IGNITION=`cat install_dir/worker.ign`

    2. Create the *.yaml* file for the worker nodes deployment.

           cat <<EOF >06_worker.yaml
           imports:
           - path: 06_worker.py

           resources:
           - name: 'worker-0' 
             type: 06_worker.py
             properties:
               infra_id: '${INFRA_ID}' 
               zone: '${ZONE_0}' 
               compute_subnet: '${COMPUTE_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '100'
               service_account_email: '${WORKER_SERVICE_ACCOUNT}' 
               ignition: '${WORKER_IGNITION}' 
           - name: 'worker-1'
             type: 06_worker.py
             properties:
               infra_id: '${INFRA_ID}' 
               zone: '${ZONE_1}' 
               compute_subnet: '${COMPUTE_SUBNET}' 
               image: '${CLUSTER_IMAGE}' 
               machine_type: 'e2-standard-4' 
               root_volume_size: '100'
               service_account_email: '${WORKER_SERVICE_ACCOUNT}' 
               ignition: '${WORKER_IGNITION}' 
           EOF

     3. Run the command to create the worker nodes.

            gcloud deployment-manager deployments create ${INFRA_ID}-worker --config 06_worker.yaml

35. With the node deployments created, there should be 2 CSRs in a pending state, we need to approve these. Once we approve the first 2, there will be 2 additional CSRs generated by those nodes, therefore, 4 CSRs (in sequence of 2 CSRs each) in total that we must approve.

        oc get csr
        oc adm certificate approve <csr name>

36. Now if we run `oc get nodes`, we should be able to see the worker nodes too. 

