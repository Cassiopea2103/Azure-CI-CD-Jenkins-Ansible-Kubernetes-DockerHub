# Deploying to K8s using Jenkins and Ansible

<div style="display:flex;flex-direction:column">
    <div style="flex:1">
        <img src="../assets/azure.png" style="border-radius:5px"/>
    </div>
    <div>
        <img src = "../assets/docker.png" width="25%" height="100"/>
        <img src = "../assets/k8s.png" width="30%" height="100"/>
        <img src = "../assets/jenkins.png" width="26%" height="100"/>
        <img src = "../assets/ansible.png" width="14%" height="100"/>
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
<img src='../azure ci_cd/0.pipeline.png' style='border-radius:5px' />

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
![Azure Resource Group](../azure%20ci_cd/1.ressource_group.png)
2.Chose a name for your resource group as well as a subscription plan an click create next 
![](../azure%20ci_cd/2.resource_group_created.png)

## II. Creating and setting up a Jenkins VM
1. The Azure portal dashboard should look like this:
![Azure dashboard](./3.azure_dashboard.png)
2. Next affect it to the resource group you just created above and give it a name like jenkins-server.  
Select the appropriate options like the region, the size, the OS and create the VM.  
<b>IMPORTANT : chose to create the VM with a new secret key file and save it securely somewhere on your computer . <i>It will be used for future connexion to the jenkins VM server.</i></b>
3. After completion of creation , the VM pannel is like : 
![jenkins-server-dashboard](../azure%20ci_cd/4.vm_jenkins_server_dashboard.png)
  
## III. Connecting to Jenkins server from local computer with MobaXterm on SSH 
_To connect the the jenkins server from our local computer and configure it , we will need to SSH to it . For this purpose, we use MobaXterm_

>To download MobaXterm , follow the link below :   
https://mobaxterm.mobatek.net

1. On the top menu interface of MobaXterm , choose the option SSH and fill the connection settings 
![SSH settings](../azure%20ci_cd/5.session_settings.png)
* Remote host is the public IP of the jenkins server you created . Copy its address from the Azure portal and paste it there.
* Username by default is set to _azureuser_ for Ubuntu LTS 22.04 version OS VM . 
* You need to use the secret key you downloaded to connect to your VM instance . For that :
  * click on "Advanced SSH settings" 
![Advanced SSH settings](../azure%20ci_cd/6.xterm_interface.png)
  * navigate to where you downloaded the secret key file and select it .
  
* Once all of the above steps completed , you can click on the OK button to connect to your jenkins server . The session SSH interface will look like :
![](../azure%20ci_cd/7.session_start_jenkins_server.png)

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
    ![java installation check](../azure%20ci_cd/8.java_path_installation.png)
    _Copy the jdk installation path_
4. Next we need to register the java installation path as an environment. 
    ```bash
    sudo nano /etc/environment
    ```

5. On a new line , create the key JAVA_HOME and affect it to the value of the JDK path 
    ![JAVA_HOME](../azure%20ci_cd/8.2.adding_java_home_env.png)
6. Force Ubuntu to reload the environment file : 
   ```bash
   source /etc/environment
   ```
7. Check the JAVA_HOME is loaded porperly : 
    ![JAVA_HOME check](../azure%20ci_cd/8.3.checking_java_env.png)

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
    ![jenkins service status](../azure%20ci_cd/8.4.jenkins_status.png)


## VI. Extra configurations on Jenkins server to enable connection to Jenkins
_Now that Jenkins is installed successfully on our VM , we will need few extra steps to connect to it and access its dashboard._ 
* By default, Jenkins runs on PORT 8080 . Access the networking configuration interface of the VM and create port rule :
![jenkins networking config](../azure%20ci_cd/9.jenkins_vm_networking0.png)
* Add port 8080 as destination range for PORT 8080 and click on "Add"
![PORT rule configuration](../azure%20ci_cd/9.jenkins_vm_networking1_port_setup.png)

## VII. Accessing Jenkins
To access Jenkins , copy the public IP address of the Jenkins server on Azure and add the port 8080
![jenkins connection](../azure%20ci_cd/10.jenkins_connection_dashboard.png)

Once done , Jenkins will ask you for the default admin user created password : 
![Jenkins password admin auth](../azure%20ci_cd/11.jenkins_interface_init.png)
_Notice the ```var/lib/secrets/initialAdminPassword``` path_

Open the file ```initialAdminPassword``` , copy it , paste it and click on "Continue".

Once connected , the Jenkins interface will be presented like : 
![jenkins Dashboard](../azure%20ci_cd//12.jenkins_dashboard.png)

## VIII. Configuring and installing Ansible
<small>_For all the automation processes in this project , Ansible will be used. It can lauch build of Docker image , publish them to docker hub, connet to K8s server to launch deployments ..._</small>
1. Create an Azure VM .  
Give it a name like Ansible server , affect it to the resource group created chose a size for it...
2. After creation , we have : 
![Azure dashboard](../azure%20ci_cd/13.azure_dashboard.png)
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
    ![Ansible installation check](../azure%20ci_cd/14.ansible_installation-check.png)

## IX : Integrating Ansible with Jenkins
<small>_The goal of doing this , is to use jenkins to trigger ansible playbook that will build docker image , publish them to Docker Hub and start K8s deployments_</small>

1. On jenkins dashboard , go to `administer jenkins>plugins>available plugins`
2. Search for SSH and check the package `publish over SSH` then install it
![Publish over SSH package](../azure%20ci_cd/15.ansible_integration_w_jenkins.png)
3. Next on Jenkins administration dahsboard go to `system configuration` and at the very bottom , you will see the newly `publish over SSH` configuration section
4. Configure there the remote connection to the Ansible server 
![Ansible remote server connection](../azure%20ci_cd/16.ansible_integration_on_jenkins_publish_over_ssh.png)
For the setting fileds : 
    * Chose a name for the remote Ansible server connection 
    * For hostname , give the public IP address of the Ansible server 
    * For the username , give the newly created user username on the Ansible server
    * Check password authentication and give the password you configured the ansibleadmin user with.
  
5. Testing connection to Ansible VM from Jenkins server : 
![Ansible remote connection test](../azure%20ci_cd/17.test_connexion_ansible_server_from_jenkins.png)

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
    ![docker folder ownership](../azure%20ci_cd/18.1.docker_folder_ownership.png)

3. Install docker package 
    ```bash
    sudo apt install docker.io
    ```
4. Check the docker status : 
    ```bash
    sudo systemctl status docker 
    ```
    ![docker installation check](../azure%20ci_cd/18.2.docker_installation_status.png)
5. Add the ansibleadmin user to dockergroup 
    ```bash
    sudo usermod -aG docker ansibleadmin
    ```
6. Next we can check the groups the use ansibleadmin belongs to 
    ```bash
    id ansibleadmin
    ```
    ![ansibleadmin user groups](../azure%20ci_cd/18.3.ansibleadmin_user_groups_check.png)


## XI.Ansible playbook for creating and publishing the Docker image
1. Retrieve the IP address of the Ansible VM for adding it to ansible hosts   
<small>_For that , install net-tools with_</small>
    ```bash
    sudo apt install net-tools
    ```
    <small>_Next we use the `ifconfig` command to retrieve the IP_</small>
    ![Ansible VM IP address](../azure%20ci_cd/19.0.ansible_vm_ip_address.png)

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
    ![Ansible playbook check](../azure%20ci_cd/19.1.ansible-playbook_check.png)
7. If all is good, we run next the playbook 
    ```ansible
    ansible-playbook cafe-app.yml
    ```
    ![ansible playbook run](../azure%20ci_cd/19.2.ansible_playbook_run.png)
8. Check if Docker image was created 
    ```bash
    docker images
    ```

    ![docker images check](../azure%20ci_cd/19.3.repo_cloned_docker_image_created.png)

9.  Check if Docker image was published on Docker Hub 
    ![docker hub image publish](../azure%20ci_cd/19.4.docker_hub_cafe_app_image.png)


## XII. Jenkins CI pipeline for lauching the Ansible playbook 
1. Create a freestyle job on jenkins with a name cafe-app-CI
2. On git section 
   *   select github repository 
   *   select the branch where the code is hosted 
3. On the section `post build actions` , select send built artifacts over SSH 
    * Select the ansible-server configured before 
    * for exec command box , fill `ansible-playbook /opt/docker/cafe-appp.yml` and save      
  
![jenkins CI config](../azure%20ci_cd/20.1.jenkins_job_config.png)
![Jenkins CI config](../azure%20ci_cd/20.2.jenkins_job_config2.png)
1. Click on `Build now` on Jenkins interface to launch the job and trigger ansible playbook for creating the docker image , pushing it to dockerhub.
    ![jenkins job build](../azure%20ci_cd/20.jenkins_job_success.png)
2. Back to the Ansible server , check for docker images 
    ![docker images](../azure%20ci_cd/21.jenkins_trigger_ansible_playbook_build_docker_images.png)

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
    ![Azure CLI installation check](../azure%20ci_cd/22.azure_cli_installation_check.png)
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
    ![K8s cluster dashboard](../azure%20ci_cd/23.k8s_cluster_dashboard.png)
10. A connexion pannel will open , copy the credentials and paste them on the terminal of the K8s server 
    ![K8s cluster connextion credentials](../azure%20ci_cd/24.connexion_to_k8s_cluster_from_azcli.png)
11. Check connection to K8s cluster by using `kubectl` command 
    ![K8s cluster connection check](../azure%20ci_cd/25.k8s_cluster_connexion_check.png)
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
    ![sshd_config file content](../azure%20ci_cd/26.k8s_server_password_auth_config.png)

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
    ![Ansible SSH to root on K8s server](../azure%20ci_cd//27.ssh_to_k8s_server_from_ansible_server.png)  
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
    ![K8s-resources-before](../azure%20ci_cd/28.k8s_resources_check0.png)
11. After playbook run and applying deployments : 
    ![K8s-resources-before](../azure%20ci_cd/28.k8s_resources_check1.png)

## XV. Preview of the application 
We copy the external IP address of the service and paste it on a web browser 
![application preview](../azure%20ci_cd/29.application_deployed_preview.png)

## XVI. Jenkins CD job 
<small>_All the deployment was done manually with Ansible to automate configurations of K8s. Now we use jenkins to include the CD job in our pipeline_</small>

1. Create a jenkins job and name it something like app_job_CD 
2. Go to `post build actions` and select `publish built artifacts over SSH`. 
3. Select the Ansible server and for the exec command , fill in the command to launch the k8s deployment playbook   
   
![jenkins CD job](../azure%20ci_cd/30.0.jenkins_job_config.png)

<b>The CD job needs to take effect after the CI job is done.</b>

For that, in post build actions , add another action `build project ( builds after )` and select the CD job
![CI-CD job complete](../azure%20ci_cd/30.1.integrate_cd_job_to_take_place_after_ci.png)


<h1><i>That was all for this project which consisted of deploying an application to Azure cloud environment using Docker, Kubernetes,  Jenkins & Ansible </i></h1>