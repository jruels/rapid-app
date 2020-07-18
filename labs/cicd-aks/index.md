# Deploy from GitHub to Azure Kubernetes Service using Jenkins

This tutorial deploys a sample application from both GitHub to an 
Azure Kubernetes Service (AKS) cluster by setting up continuous integration (CI) and 
continuous deployment (CD) in Jenkins. That way, when you 
update your app by pushing commits to GitHub, Jenkins 
automatically runs a new container build, pushes container 
images to Azure Container Registry (ACR), and then runs your app in AKS. 

In this tutorial, you'll complete these tasks:

  * Deploy a sample Azure vote app to an AKS cluster.
  * Create a basic Jenkins project.
  * Set up credentials for Jenkins to interact with ACR.
  * Create a Jenkins build job and GitHub webhook for automated builds.
  * Test the CI/CD pipeline to update an application in AKS based on GitHub code commits.

## Prepare your app

In this article, you use a sample Azure vote application that contains a web interface hosted in one or more pods, and a second pod hosting Redis for temporary data storage. Before you integrate Jenkins and AKS for automated deployments, first manually prepare and deploy the Azure vote application to your AKS cluster. This manual deployment is version one of the application, and lets you see the application in action.

Create a free GitHub account if you do not already have one.   
https://github.com/join

Fork the following GitHub repository for the sample application - https://github.com/Azure-Samples/azure-voting-app-redis  
To fork the repo to your own GitHub account, select the **Fork** button in the top right-hand corner.

Clone the fork to your development system. Make sure you use the URL of your fork when cloning this repo:

```console
git clone https://github.com/<your-github-account>/azure-voting-app-redis.git
```

Change to the directory containing the application:

```console
cd azure-voting-app-redis/azure-vote
```

Set an environment variable for Azure Container Registry. The registry name must be unique. Use the following variable replacing `<your initials>`

Set an environment variable
```console
ACR=voting<your initials>
```

Create a new resource group
```console
az group create --name voting --location westus
```

Create an Azure Container Registry for storing images.
```console
az acr create -n $ACR -g voting --sku Standard
```
Now use the following command to find the Registry login server:
```console
az acr list --resource-group voting --query "[].{acrLoginServer:loginServer}" --output table
```

You should see something like this 
```
AcrLoginServer
--------------------
votingjrs.azurecr.io
```

Create images and push to new registry. The '.' at the end tells the builder to look in current directory for `Dockerfile` 
```console
az acr build -t azure-vote-front:v1 -r $ACR .
```

The output should show that the image was built and pushed successfully.
```
- image:
    registry: votingjrs.azurecr.io
    repository: azure-vote-front
    tag: v3
    digest: sha256:8156bb67c5a34f2ad5d9724ae1ef743bc0ca882156459c3ec0a5079cfb288165
  runtime-dependency:
    registry: registry.hub.docker.com
    repository: tiangolo/uwsgi-nginx-flask
    tag: python3.6
    digest: sha256:98970564aeb7b5e636a0411936e61c02e81b19d5f93a4cd01715b6a690e6790b
  git: {}

Run ID: cc3 was successful after 33s
```

## Integrate Kubernetes and ACR
The AKS cluster we have been using does not have access to Azure Container Registry. Run the following command to authorize it.

```console
az aks update -n myAKSCluster -g voting --attach-acr $ACR
```

## Deploy the sample application to AKS

To deploy the sample application to your AKS cluster, you can use the Kubernetes manifest file in the root of the Azure vote repository repo. Open the *azure-vote-all-in-one-redis.yaml* manifest file with an editor such as `vi`. Replace `microsoft` with your ACR login server name. This value is found on line **47** of the manifest file:

```yaml
containers:
- name: azure-vote-front
  image: microsoft/azure-vote-front:v1
```

Here is an example of what it should look like after you add your registry:

```yaml
containers:
- name: azure-vote-front
  image: votingjrs.azurecr.io/azure-vote-front:v1
```

Next, use the kubectl apply command to deploy the application to your AKS cluster:

```console
kubectl apply -f azure-vote-all-in-one-redis.yaml
```

A Kubernetes load balancer service is created to expose the application to the internet. This process can take a few minutes. To monitor the progress of the load balancer deployment, use the kubectl get service command with the `--watch` argument. Once the *EXTERNAL-IP* address has changed from *pending* to an *IP address*, use `Control + C` to stop the kubectl watch process.

```console
$ kubectl get service azure-vote-front --watch

NAME               TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
azure-vote-front   LoadBalancer   10.0.215.27   <pending>     80:30747/TCP   22s
azure-vote-front   LoadBalancer   10.0.215.27   40.117.57.239   80:30747/TCP   2m
```

To see the application in action, open a web browser to the external IP address of your service. The Azure vote application is displayed, as shown in the following example:

![Azure sample vote application running in AKS](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/azure-vote.png)



## Deploy a Jenkins container to an Azure VM

To quickly deploy Jenkins for use in this article, you can use the following script to deploy an Azure virtual machine, configure network access, and complete a basic installation of Jenkins. For authentication between Jenkins and the AKS cluster, the script copies your Kubernetes configuration file from your development system to the Jenkins system.

**WARNING**
This sample script is for demo purposes to quickly provision a Jenkins environment that runs on an Azure VM. It uses the Azure custom script extension to configure a VM and then display the required credentials. Your *~/.kube/config* is copied to the Jenkins VM.

Run the following commands to download and run the script. 

```console
curl https://gist.githubusercontent.com/jruels/dd44923beb9e4111eb13ad4cd204b1d2/raw/2bb83570d4f26573d753d0d5d625acab25826ff8/azure-jenkins-vm.sh > azure-jenkins.sh
sh azure-jenkins.sh
```

It takes a few minutes to create the VM and deploy the required components for Docker and Jenkins. When the script has completed, it outputs an address for the Jenkins server and a key to unlock the dashboard, as shown in the following example output:

```
Open a browser to http://40.115.43.83:8080
Enter the following to Unlock Jenkins:
667e24bba78f4de6b51d330ad89ec6c6
```

Open a web browser to the URL displayed and enter the unlock key. Follow the on-screen prompts to complete the Jenkins configuration:

- Choose **Install suggested plugins**
- Create the first admin user. Enter a username, such as *azureuser*, then provide your own secure password. Finally, type a full name and e-mail address.
- Select **Save and Finish**
- Once Jenkins is ready, select **Start using Jenkins**
- Sign in to Jenkins with the username and password you created in the install process.

## Create a Jenkins environment variable

A Jenkins environment variable is used to hold the ACR login server name. This variable is referenced during the Jenkins build job. To create this environment variable, complete the following steps:

- On the left-hand side of the Jenkins portal, select **Manage Jenkins** > **Configure System**
- Under **Global Properties**, select **Environment variables**. Add a variable with the name `ACR_LOGINSERVER` and the value of your ACR login server.

    ![Jenkins environment variables](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/env-variables.png)

- When complete, click **Save** at the bottom of the Jenkins configuration page.

## Create a Jenkins credential for ACR

To allow Jenkins to build and then push updated container images to ACR, you need to specify credentials for ACR. This authentication can use Azure Active Directory service principals. During the CI/CD process, Jenkins builds new container images based on application updates, and needs to then *push* those images to the ACR registry. Configure a service principal for Jenkins with *Contributor* permissions to your ACR registry.

### Create a service principal for Jenkins to use ACR

First, create a service principal using the az ad sp create-for-rbac command:

```azurecli
$ az ad sp create-for-rbac --skip-assignment

{
  "appId": "626dd8ea-042d-4043-a8df-4ef56273670f",
  "displayName": "azure-cli-2018-09-28-22-19-34",
  "name": "http://azure-cli-2018-09-28-22-19-34",
  "password": "1ceb4df3-c567-4fb6-955e-f95ac9460297",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db48"
}
```

Make a note of the *appId* and *password* shown in your output. These values are used in following steps to configure the credential resource in Jenkins.

Get the resource ID of your ACR registry using the az acr show command, and store it as a variable. Provide your resource group name and ACR name:

```azurecli
ACR_ID=$(az acr show --resource-group voting --name <acrLoginServer> --query "id" --output tsv)
```

Now create a role assignment to assign the service principal *Contributor* rights to the ACR registry. In the following example, provide your own *appId* shown in the output a previous command to create the service principal:

```azurecli
az role assignment create --assignee 626dd8ea-042d-4043-a8df-4ef56273670f --role Contributor --scope $ACR_ID
```

### Create a credential resource in Jenkins for the ACR service principal

With the role assignment created in Azure, now store your ACR credentials in a Jenkins credential object. These credentials are referenced during the Jenkins build job.

Back on the left-hand side of the Jenkins portal, click **Credentials** > **Jenkins** > **Global credentials (unrestricted)** > **Add Credentials**

Ensure that the credential kind is **Username with password** and enter the following items:

- **Username** - The *appId* of the service principal created for authentication with your ACR registry.
- **Password** - The *password* of the service principal created for authentication with your ACR registry.
- **ID** - Credential identifier such as *acr-credentials*

When complete, the credentials form looks like the following example:

![Create a Jenkins credential object with the service principal information](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/acr-credentials.png)

Click **OK** and return to the Jenkins portal.

## Create a Jenkins project

From the home page of your Jenkins portal, select **New item** on the left-hand side:

1. Enter *azure-vote* as job name. Choose **Freestyle project**, then select **OK**
1. Under the **General** section, select **GitHub project** and enter your forked repo URL, such as *https:\//github.com/\<your-github-account\>/azure-voting-app-redis*
1. Under the **Source code management** section, select **Git**, enter your forked repo *.git* URL, such as *https:\//github.com/\<your-github-account\>/azure-voting-app-redis.git*

1. Under the **Build Triggers** section, select **GitHub hook trigger for GITscm polling**
1. Under **Build Environment**, select **Use secret texts or files**
1. Under **Bindings**, select **Add** > **Username and password (separated)**
   - Enter `ACR_ID` for the **Username Variable**, and `ACR_PASSWORD` for the **Password Variable**

     ![Jenkins bindings](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/bindings.png)

1. Choose to add a **Build Step** of type **Execute shell** and use the following text. This script builds a new container image and pushes it to your ACR registry.

    ```bash
    # Build new image and push to ACR.
    WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${GIT_COMMIT}-${BUILD_NUMBER}"
    docker build -t $WEB_IMAGE_NAME ./azure-vote
    docker login ${ACR_LOGINSERVER} -u ${ACR_ID} -p ${ACR_PASSWORD}
    docker push $WEB_IMAGE_NAME
    ```

1. Add another **Build Step** of type **Execute shell** and use the following text. This script updates the application deployment in AKS with the new container image from ACR.

    ```bash
    # Update kubernetes deployment with new image.
    WEB_IMAGE_NAME="${ACR_LOGINSERVER}/azure-vote-front:kube${GIT_COMMIT}-${BUILD_NUMBER}"
    /var/jenkins_home/kubectl set image deployment/azure-vote-front azure-vote-front=$WEB_IMAGE_NAME --kubeconfig /var/jenkins_home/config
    ```

1. Once completed, click **Save**.

## Test the Jenkins build

Before you automate the job based on GitHub commits, first manually test the Jenkins build. This manual build validates that the job has been correctly configured, the proper Kubernetes authentication file is in place, and that the authentication with ACR works.

On the left-hand menu of the project, select **Build Now**.

![Jenkins test build](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/test-build.png)

The first build takes a minute or two as the Docker image layers are pulled down to the Jenkins server. Subsequent builds can use the cached image layers to improve the build times.

During the build process, the GitHub repository is cloned to the Jenkins build server. A new container image is built and pushed to the ACR registry. Finally, the Azure vote application running on the AKS cluster is updated to use the new image. Because no changes have been made to the application code, the application is not changed if you view the sample app in a web browser.

Once the build job is complete, click on **build #1** under build history. Select **Console Output** and view the output from the build process. The final line should indicate a successful build.

## Create a GitHub webhook

With a successful manual build complete, now integrate GitHub into the Jenkins build. A webhook can be used to run the Jenkins build job each time a code commit is made in GitHub. To create the GitHub webhook, complete the following steps:

1. Browse to your forked GitHub repository in a web browser.
1. Select **Settings**, then select **Webhooks** on the left-hand side.
1. Choose to **Add webhook**. For the *Payload URL*, enter `http://<publicIp:8080>/github-webhook/`, where `<publicIp>` is the IP address of the Jenkins server. Make sure to include the trailing /. Leave the other defaults for content type and to trigger on *push* events.
1. Select **Add webhook**.

    ![Create a GitHub webhook for Jenkins](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/webhook.png)

## Test the complete CI/CD pipeline

Now you can test the whole CI/CD pipeline. When you push a code commit to GitHub, the following steps happen:

1. The GitHub webhook reaches out to Jenkins.
1. Jenkins starts the build job and pulls the latest code commit from GitHub.
1. A Docker build is started using the updated code, and the new container image is tagged with the latest build number.
1. This new container image is pushed to Azure Container Registry.
1. Your application deployed to Azure Kubernetes Service updates with the latest container image from the Azure Container Registry registry.

On your development machine, open up the cloned application with a code editor. Under the */azure-vote/azure-vote* directory, open the file named **config_file.cfg**. Update the vote values in this file to something other than cats and dogs, as shown in the following example:

```
# UI Configurations
TITLE = 'Azure Voting App'
VOTE1VALUE = 'Blue'
VOTE2VALUE = 'Purple'
SHOWHOST = 'false'
```

When updated, save the file, commit the changes, and push these to your fork of the GitHub repository. The GitHub webhook triggers a new build job in Jenkins. In the Jenkins web dashboard, monitor the build process. It takes a few seconds to pull the latest code, create and push the updated image, and deploy the updated application in AKS.

Once the build is complete, refresh your web browser of the sample Azure vote application. Your changes are displayed, as shown in the following example:

![Sample Azure vote in AKS updated by the Jenkins build job](https://docs.microsoft.com/en-us/azure/developer/jenkins/media/deploy-from-github-to-aks/azure-vote-updated.png)


## Lab Complete
