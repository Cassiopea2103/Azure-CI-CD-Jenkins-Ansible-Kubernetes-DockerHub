# Deploying to K8s using Jenkins and Ansible

<div style="display:flex;flex-direction:column">
    <div style="flex:1">
        <img src="https://github.com/user-attachments/assets/c150db1c-39bf-4694-8104-a620bfa72611" style="border-radius:5px"/>
    </div>
    <div>
        <img src = "https://github.com/user-attachments/assets/c03691f2-f9c1-45b7-8a48-62ace257bee2" width="25%" height="150"/>
        <img src = "https://github.com/user-attachments/assets/192c02bd-6845-443b-99db-3376145dcee7" width="30%" height="150"/>
        <img src = "https://github.com/user-attachments/assets/9a8ffd11-11ba-4b86-8ea3-1568c986733c" width="26%" height="150"/>
        <img src = "https://github.com/user-attachments/assets/db93abe9-69b1-4edb-b809-4d0f2202ad13" width="14%" height="150"/>
    </div>
</div>

---

## Project Overview 
In this project , we build a basic static website that is pushed then to Github.  
We want to automate the build of the project as well as its deployment to Kubernetes on an Azure cloud environment.  
  
Here is a breakdown of the different steps on how the flow goes : 
1. The user does a push of his code to github 
2. Jenkins detects the push and start the CI pipeline
3. The CI pipeline uses an Ansible server to build the Docker image  with a Dockerfile included in the code 
4. The same CI pipeline then publishes the built Docker image to Docker Hub for later use in this project.
5. The the Ansible server uses another K8s server to create deployments of the published docker images and runs it in pods in a K8s Azure cluster . 
  
Below is the schema explaining these steps :  
<img src='https://github.com/user-attachments/assets/2b063c91-2f4a-48ad-94ea-796e0c0e3bcf' style='border-radius:5px' />

---

## Configurations 
From creating virtual machines on Azure to setting them up for appropriate use , this project includes many configurations to do . 
In this section , we will see :
* how to create the Jenkins server and configure it 
* how to install Jenkins on the Azure VM 
* how to install Ansible and configure it 
* how to install the K8s server and connect it to an Azure K8s cluster 
* establishing connexion between the Jenkins serer and Ansible server 
* connecting Ansible server to K8s server 
* Jenkins pipelines , Ansible playbooks as well as K8s deployments files .

---
## I. Creating an Azure resource group 
_Prerequisites : You have to create before any of this a Microsoft Azure account and setup up a subscription in order to use its different services_

1. Open the Azure portal and create an overall resource group to contain all of our Azure resources for future use ( VMs and clusters of Kubernetes ) 
<img width="932" alt="1 ressource_group" src="https://github.com/user-attachments/assets/4089f613-da15-484b-a475-55c879eaba6f">

2.Chose a name for your resource group as well as a subscription plan an click create next 
<img width="789" alt="2 resource_group_created" src="https://github.com/user-attachments/assets/f693db36-b426-49ea-a842-05e7f0d1025c">
## II. Creating and setting up a Jenkins VM
1. The Azure portal dashboard should look like this:
<img width="838" alt="3 azure_dashboard" src="https://github.com/user-attachments/assets/303708a8-0afb-4f55-8599-9861002dde19">

2. Next affect it to the resource group you just created above and give it a name like jenkins-server.  
Select the appropriate options like the region, the size, the OS and create the VM.  
<b>IMPORTANT : chose to create the VM with a new secret key file and save it securely somewhere on your computer . <i>It will be used for future connexion to the jenkins VM server.</i></b>
3. After completion of creation , the VM pannel is like : 
<img width="949" alt="4 vm_jenkins_server_dashboard" src="https://github.com/user-attachments/assets/20f54dd6-a713-4934-b0ea-93cd0c084bc8">

  
## III. Connecting to Jenkins server from local computer with MobaXterm on SSH 
_To connect the the jenkins server from our local computer and configure it , we will need to SSH to it . For this purpose, we use MobaXterm_

>To download MobaXterm , follow the link below :   
https://mobaxterm.mobatek.net

1. On the top menu interface of MobaXterm , choose the option SSH and fill the connection settings 
<img width="635" alt="5 session_settings" src="https://github.com/user-attachments/assets/7afd2545-bfd4-4631-9c82-631c2b073d91">

* Remote host is the public IP of the jenkins server you created . Copy its address from the Azure portal and paste it there.
* Username by default is set to _azureuser_ for Ubuntu LTS 22.04 version OS VM . 
* You need to use the secret key you downloaded to connect to your VM instance . For that :
  * click on "Advanced SSH settings" 
<img width="636" alt="6 xterm_interface" src="https://github.com/user-attachments/assets/d7f429b8-0419-421e-9543-a500c4e17512">

  * navigate to where you downloaded the secret key file and select it .
  
* Once all of the above steps completed , you can click on the OK button to connect to your jenkins server . The session SSH interface will look like :
<img width="676" alt="7 session_start_jenkins_server" src="https://github.com/user-attachments/assets/cf8727db-a6bc-42fb-a2af-786b7181ae45">


## IV. Installing JAVA JDK
_Jenkins needs a JDK to run properly_

1. Update the packages : 
    ```bash
    sudo apt update
    ```
2. Install a JDK  ( jenkins need a jdk to run ) :
    ```bash
    sudo apt install default-jdk
    ```
3. Check for the jdk installation on the system : 
    ```bash
    update-alternatives --config java
    ```
   <img width="683" alt="8 java_path_installation" src="https://github.com/user-attachments/assets/05861c4b-e859-406f-9601-3c1be05810b3">

    _Copy the jdk installation path_
4. Next we need to register the java installation path as an environment. 
    ```bash
    sudo nano /etc/environment
    ```

5. On a new line , create the key JAVA_HOME and affect it to the value of the JDK path 
    <img width="724" alt="8 2 adding_java_home_env" src="https://github.com/user-attachments/assets/91369604-d635-4b05-b2c1-7682370a1337">

6. Force Ubuntu to reload the environment file : 
   ```bash
   source /etc/environment
   ```
7. Check the JAVA_HOME is loaded porperly : 
    <img width="298" alt="8 3 checking_java_env" src="https://github.com/user-attachments/assets/5e64f60f-f239-41cf-842b-477c55336307">


## V. Installing Jenkins
<small>_Jenkins is our pipeline orchestration tool for this project. It will help detect a code push a trigger a CI job for building Docker image that will be pushed to Docker Hub. Later , il will launch also a CD job for creating deployment of the container of pods._</small>
> To install jenkins on Ubuntu , follow the official guide below :   
https://www.jenkins.io/doc/book/installing/linux/

* You can then check for your jenkins installation 
    ```bash
    cat systemctl jenkins
    ```

* Next enable and start jenkins server 
    ```bash
    sudo systemctl enable jenkins
    sudo systemctl start jenkins
    ```

* Check the Jenkins service status : 
    ```bash
    sudo systemctl status jenkins
    ```
   <img width="770" alt="8 4 jenkins_status" src="https://github.com/user-attachments/assets/81c9d1d3-86b3-41e5-8f9b-d2289badc5d5">



## VI. Extra configurations on Jenkins server to enable connection to Jenkins
_Now that Jenkins is installed successfully on our VM , we will need few extra steps to connect to it and access its dashboard._ 
* By default, Jenkins runs on PORT 8080 . Access the networking configuration interface of the VM and create port rule :
<img width="941" alt="9 jenkins_vm_networking0" src="https://github.com/user-attachments/assets/20359a8b-7e8e-4566-b79a-b51201089c6c">

* Add port 8080 as destination range for PORT 8080 and click on "Add"
<img width="427" alt="9 jenkins_vm_networking1_port_setup" src="https://github.com/user-attachments/assets/bfe3d1d9-7b8a-4e91-aad4-df65343793af">


## VII. Accessing Jenkins
To access Jenkins , copy the public IP address of the Jenkins server on Azure and add the port 8080
<img width="563" alt="10 jenkins_connection_dashboard" src="https://github.com/user-attachments/assets/a283c535-7f8d-4653-877e-fcc5f249e896">


Once done , Jenkins will ask you for the default admin user created password : 
<img width="736" alt="11 jenkins_interface_init" src="https://github.com/user-attachments/assets/f6f15f55-879c-4eb1-9eea-2e24b29ad825">

_Notice the ```var/lib/secrets/initialAdminPassword``` path_

Open the file ```initialAdminPassword``` , copy it , paste it and click on "Continue".

Once connected , the Jenkins interface will be presented like : 
<img width="947" alt="12 jenkins_dashboard" src="https://github.com/user-attachments/assets/f7c2af5f-4542-48f5-acf0-5fe09b4801d2">


## VIII. Configuring and installing Ansible
<small>_For all the automation processes in this project , Ansible will be used. It can lauch build of Docker image , publish them to docker hub, connet to K8s server to launch deployments ..._</small>
1. Create an Azure VM .  
Give it a name like Ansible server , affect it to the resource group created chose a size for it...
2. After creation , we have : 
<img width="933" alt="13 azure_dashboard" src="https://github.com/user-attachments/assets/3097f4ce-57b6-42f1-9e66-42217bf7188a">

3. On the VM network configuration interface , add PORT inbound rules for PORT ranges 8080-8090 
4. add new user to the system for ansible :
    ```bash
    sudo adduser ansibleadmin
    ```
    ```setup a password for that user```

5. add the ansibleadmin user to sudo users : 
    ```bash
    sudo usermod -aG sudo ansibleadmin
    ```
6. Restart the SSH connexion to the Ansible server :
    ```bash
    sudo init 6
    ```
7. Reconnect to the Ansible server VM using the newly created ansibleadmin user
8. Generate a SSH key for the Ansible server :
    ```bash
    ssh-keygen
    ```
    The generated key has 2 roles : 
    * It can be used to publish build artifacts since the Jenkins server
    * It will help establish a self-trust ( loopback ) to build docker images using the Ansible server itself.
9. Now add the Ansible repository to the server 
    ```bash
    sudo apt-add-repository ppa:ansible/ansible
    ```
10. Update system packages : 
    ```bash
    sudo apt update
    ```
11. Install Ansible package on the server 
    ```bash
    sudo apt install ansible-core
    ```
12. Check for Ansible installation 
    ```bash
    ansible --version
    ``` 
    <img width="754" alt="14 ansible_installation-check" src="https://github.com/user-attachments/assets/917499b6-92cc-4b3d-b316-28d24df3e8a6">


## IX : Integrating Ansible with Jenkins
<small>_The goal of doing this , is to use jenkins to trigger ansible playbook that will build docker image , publish them to Docker Hub and start K8s deployments_</small>

1. On jenkins dashboard , go to `administer jenkins>plugins>available plugins`
2. Search for SSH and check the package `publish over SSH` then install it
<img width="926" alt="15 ansible_integration_w_jenkins" src="https://github.com/user-attachments/assets/faa2674e-6ec0-4dee-868f-ac56b1d58aee">

3. Next on Jenkins administration dahsboard go to `system configuration` and at the very bottom , you will see the newly `publish over SSH` configuration section
4. Configure there the remote connection to the Ansible server 
<img width="835" alt="16 ansible_integration_on_jenkins_publish_over_ssh" src="https://github.com/user-attachments/assets/ce960151-19b7-446b-8892-0340cd379494">

For the setting fileds : 
    * Chose a name for the remote Ansible server connection 
    * For hostname , give the public IP address of the Ansible server 
    * For the username , give the newly created user username on the Ansible server
    * Check password authentication and give the password you configured the ansibleadmin user with.
  
5. Testing connection to Ansible VM from Jenkins server : 
<img width="803" alt="17 test_connexion_ansible_server_from_jenkins" src="https://github.com/user-attachments/assets/a0cf5816-0120-4f93-b9f7-5e634c6d98d5">


## X. Installing Docker on Ansible server 
1. Create a docker folder at `/opt` on the Ansible server 
    ```bash
    cd /opt
    mkdir docker
    ```

2. By default , the `root user` is the default owner of all folder/files on the system.  
Grant ansibleadmin user exlusive administration on the docker folder 
    ```bash
    sudo chown -R ansibleadmin:ansibleadmin docker
    ```
    <img width="517" alt="18 1 docker_folder_ownership" src="https://github.com/user-attachments/assets/0471a741-a78a-4056-8ee9-5c6f852c9df8">


3. Install docker package 
    ```bash
    sudo apt install docker.io
    ```
4. Check the docker status : 
    ```bash
    sudo systemctl status docker 
    ```
    <img width="766" alt="18 2 docker_installation_status" src="https://github.com/user-attachments/assets/d54fe469-78bd-47f4-bc5d-990cceb65fe5">

5. Add the ansibleadmin user to dockergroup 
    ```bash
    sudo usermod -aG docker ansibleadmin
    ```
6. Next we can check the groups the use ansibleadmin belongs to 
    ```bash
    id ansibleadmin
    ```
    <img width="578" alt="18 3 ansibleadmin_user_groups_check" src="https://github.com/user-attachments/assets/e5dd0520-3883-4c1d-b064-427d56b61fb0">



## XI.Ansible playbook for creating and publishing the Docker image
1. Retrieve the IP address of the Ansible VM for adding it to ansible hosts   
<small>_For that , install net-tools with_</small>
    ```bash
    sudo apt install net-tools
    ```
    <small>_Next we use the `ifconfig` command to retrieve the IP_</small>
    <img width="463" alt="19 0 ansible_vm_ip_address" src="https://github.com/user-attachments/assets/8f918010-11dd-4288-9d52-231202fba3f4">


2. Navigate to ansible configuration files folder and modify the `hosts` file 
    ```bash
    cd /etc/ansible
    sudo nano hosts
    ```
3. Create a hosts block and append the Ansible server you just retrieved with the `ifconfig` command 
    > [ansible]  
    > xx.x.x.x ( replace with the Ansible server IP )

4. Create a playbook in `/opt/docker` 
    ```bash
    nano cafe-app.yml
    ```
5. Put the following content in the file ( you can also find it in the source code of the project ) 
    ```ansible
        --- 

        - hosts : ansible 

        tasks : 
        - name : clone github repository of the application 
            git : 
            repo : https://github.com/Cassiopea2103/Azure-CI-CD-Jenkins-Ansible-Kubernetes-DockerHub.git
            dest : /opt/docker/cafe-app
            clone : yes
            update : yes 

        - name : create Docker image
            command : docker build -t cafe-app:latest /opt/docker/cafe-app
            args : 
            chdir : /opt/docker


        - name : create tag for image to push on Docker Hub 
            command : docker tag cafe-app:latest cassiopea21/cafe-app:latest

        - name : push docker image
            command : docker push cassiopea21/cafe-app:latest
        
    ```

    What this playbook does : 
    * applies to the ansible block ( meaning the ansible server itselft )
    * clones the github repository 
    * creates the code docker image ( we will see it just below ) 
    * create the tag for the docker image 
    * pushes the docker image to docker hub 
      

    Content of the Dockerfile in the code : 
    ```docker
    FROM httpd:2.4
    COPY . /usr/local/apache2/htdocs/
    ```
   * copies all the code files and move it to the `apache2/htdocs` server folder

6. Next we check the ansible playbook to see if there are no errors 
    ```ansible
    ansible-playbook cafe-app.yml --check
    ```
    <img width="773" alt="19 1 ansible-playbook_check" src="https://github.com/user-attachments/assets/62a7c8e3-78c7-4aa3-804c-84f73c618aaf">

7. If all is good, we run next the playbook 
    ```ansible
    ansible-playbook cafe-app.yml
    ```
    <img width="770" alt="19 2 ansible_playbook_run" src="https://github.com/user-attachments/assets/14b1a3cc-057b-4748-bb30-11467907743e">

8. Check if Docker image was created 
    ```bash
    docker images
    ```

    <img width="409" alt="19 3 repo_cloned_docker_image_created" src="https://github.com/user-attachments/assets/11a1eb07-e765-43c0-85c6-1fc0281d5bdd">


9.  Check if Docker image was published on Docker Hub 
    <img width="935" alt="19 4 docker_hub_cafe_app_image" src="https://github.com/user-attachments/assets/2a28e355-f0fc-4e35-8aae-2a4f097d764b">



## XII. Jenkins CI pipeline for lauching the Ansible playbook 
1. Create a freestyle job on jenkins with a name cafe-app-CI
2. On git section 
   *   select github repository 
   *   select the branch where the code is hosted 
3. On the section `post build actions` , select send built artifacts over SSH 
    * Select the ansible-server configured before 
    * for exec command box , fill `ansible-playbook /opt/docker/cafe-appp.yml` and save      
  
<img width="841" alt="20 1 jenkins_job_config" src="https://github.com/user-attachments/assets/3edd89ed-299c-4583-9249-fd840bb4cf8a">

<img width="805" alt="20 2 jenkins_job_config2" src="https://github.com/user-attachments/assets/323d2e60-91d3-4b5d-bcde-ad2b6a8540c6">

1. Click on `Build now` on Jenkins interface to launch the job and trigger ansible playbook for creating the docker image , pushing it to dockerhub.
    <img width="736" alt="20 jenkins_job_success" src="https://github.com/user-attachments/assets/d8a3d8c2-9e95-4f4e-8fdd-f9aad896a665">

2. Back to the Ansible server , check for docker images 
    <img width="419" alt="21 jenkins_trigger_ansible_playbook_build_docker_images" src="https://github.com/user-attachments/assets/56d548d4-f606-42d4-b9bc-f226a148b5dd">


## XIII Configuring the K8s server and connecting to K8s Azure cluster
<small>_The Kubernetes server will host our deployment files andd will also be connected to Azure K8s cluster for manager our resources like pods..._</small>

1. Create a VM on Azure 
2. Create a K8s cluster on Azure
3. SSH to the K8s-server and enter `sudo su` to use root user 
4. Next install the Azure CLI on the K8s server by following this link :   
https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt
5. On Linux Ubuntu , install Azure CLI with this one liner : 
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
6. Check the AZ CLI installation : 
    ```bash
    az --version
    ```
    <img width="403" alt="22 azure_cli_installation_check" src="https://github.com/user-attachments/assets/a9ac78ed-bf0d-4b32-86a6-84693fb52fa1">

7. Connect to Azure portal from Azure CLI on K8s server        
    ```bash
    az login --use-device-login
    ```
8. Install K8s CLI 
    ```bash
    az aks install-cli
    ```
9. Once K8s CLI installed , we can connect to Azure K8s cluster .   
On the K8s cluster , select the `Connect` option 
    <img width="947" alt="23 k8s_cluster_dashboard" src="https://github.com/user-attachments/assets/f46a4af3-8f96-456f-b40e-d263d7f99d26">

10. A connexion pannel will open , copy the credentials and paste them on the terminal of the K8s server 
    <img width="760" alt="24 connexion_to_k8s_cluster_from_azcli" src="https://github.com/user-attachments/assets/40ff1f3f-6894-42a0-826a-050a8d4f15ab">

11. Check connection to K8s cluster by using `kubectl` command 
    <img width="455" alt="25 k8s_cluster_connexion_check" src="https://github.com/user-attachments/assets/381f0e4b-7e2a-49da-b036-0b9d64c0ab9d">

12. On K8s server , navigate to root and create the deployment file 
    ```bash
    cd ~
    nano cafe-app-deployment.yml
    ```

13. Write the K8s deployment file .   
The content should be as follow  ( file in source code of project as cafe-app-deployment.yml ):   
    ```kubernetes
    apiVersion: apps/v1 
    kind: Deployment

    metadata:
    name: cafe-app
    labels:
        app : cafe-app


    spec:
    replicas: 2
    selector:
        matchLabels:
        app : cafe-app
    template:
        metadata : 
        labels:
            app : cafe-app 
        spec:
        containers:
            - name: cafe-app 
            image: cassiopea21/cafe-app
            imagePullPolicy: Always
            ports:
                - containerPort: 80
    strategy:
        type: RollingUpdate
        rollingUpdate:
        maxSurge : 1 
        maxUnavailable : 1 
    ```

12. Creating the service for exposing the pods in the cluster : 
    ```kubernetes
    apiVersion: v1 
    kind: Service

    metadata:
    name : cafe-app-service
    labels : 
        app : cafe-app


    spec : 
    selector:
        app : cafe-app

    ports:
        - port: 80
        targetPort : 80
    type: LoadBalancer
    ```
    
## XIV. Automating K8s deployment configurations with Ansible 
<small>_For automating the task of Kubernetes deployments, we need to use Ansible and pass it the IP of the K8s server, which is connected to the K8s cluster on Azure_</small>.

1. Install net-tools on the K8s server 
    ```bash
    sudo apt install net-tools
    ```
2. Retrieve the K8s server IP address 
    ```bash
    ifconfig
    ```
3. Add the K8s server IP address to hosts of Ansible
    ```bash
    cd /etc/ansible 
    sudo nano hosts
    ```
    Apprend the K8s IP in a kubernetes group :

4. To connect to K8s server from Ansible VM , we need to enable on K8s server :
    * password authentication 
    * root login  

    For that , navigate to `/etc/ssh` 
    ```bash
    cd /etc/ssh
    sudo nano sshd_config
    ```
    Modify the file to include the below lines : 
    <img width="494" alt="26 k8s_server_password_auth_config" src="https://github.com/user-attachments/assets/263e6074-37a5-4268-9afe-1d99016a7d59">


5. Set password for K8s server : 
    ```bash
    sudo passwd root
    ```
6. Go to Ansible server and copy it ssh key to K8s server 
    ```bash
    ssh-copy-id root@xx.x.x.x ( xx.x.x.x is the IP of K8S server )
    ```
    This is principally to enable a self-trust ( or loopback ) , meaning the K8s server will add the Ansible VM to its trusted hosts , so it won't be prompted any password when trying to automate tasks on it . 
7. Next we SSH to the K8s server from the Ansible server to check
    <img width="441" alt="27 ssh_to_k8s_server_from_ansible_server" src="https://github.com/user-attachments/assets/95b7a163-fff4-4cae-ac83-9ab6d63f33ff">

    As we can see , no password was prompted to the ansibleadmin when connecting to K8s server

8. Create the ansible playbook for lauching deployments of Kubernetes on the K8s server 
    ```bash
    cd /opt/docker
    nano k8s-deployment-playbook.yml
    ```

    ```ansible
    ---

    - hosts: kubernetes
    user : root 

    tasks:
        - name : deploy cafe-app on k8s
        command : kubectl apply -f cafe-app-deployment.yml

        - name : create service for cafe-app discovery 
        command : kubectl apply -f cafe-app-service.yml

        - name : update deployment with new pods if image container changes in docker hub 
        command : kubectl rollout restart deployment.apps/cafe-app
    ```

    What this playbook does : 
    * applies to kubernetes host 
    * applies the deployment file 
    * applies the service file 
    * update deployment files rollout 

9.  We check the playbook 
    ```ansible
    ansible-playbook k8s-deployment-playbook.yml --check
    ```
    If all is good then we run the playbook
    ```ansible
    ansible-playbook k8s-deployment-playbook.yml 
    ```

10. Before applying the playbook : 
    <img width="464" alt="28 k8s_resources_check0" src="https://github.com/user-attachments/assets/b9795ae5-64d4-41a1-aca7-ab76a54e568c">

11. After playbook run and applying deployments : 
    <img width="584" alt="28 k8s_resources_check1" src="https://github.com/user-attachments/assets/22d3c9c4-2e19-4217-8f20-8ba3635766ab">


## XV. Preview of the application 
We copy the external IP address of the service and paste it on a web browser 
<img width="927" alt="29 application_deployed_preview" src="https://github.com/user-attachments/assets/b1a62637-19e9-45f2-9932-1ba115575229">


## XVI. Jenkins CD job 
<small>_All the deployment was done manually with Ansible to automate configurations of K8s. Now we use jenkins to include the CD job in our pipeline_</small>

1. Create a jenkins job and name it something like app_job_CD 
2. Go to `post build actions` and select `publish built artifacts over SSH`. 
3. Select the Ansible server and for the exec command , fill in the command to launch the k8s deployment playbook   
   
<img width="960" alt="30 0 jenkins_job_config" src="https://github.com/user-attachments/assets/23c3b337-23b6-4702-9284-d135fbafee77">


<b>The CD job needs to take effect after the CI job is done.</b>

For that, in post build actions , add another action `build project ( builds after )` and select the CD job
<img width="722" alt="30 1 integrate_cd_job_to_take_place_after_ci" src="https://github.com/user-attachments/assets/3cd380b6-919a-4067-a989-65c763d3ec9c">



<h1><i>That was all for this project which consisted of deploying an application to Azure cloud environment using Docker, Kubernetes,  Jenkins & Ansible </i></h1>
