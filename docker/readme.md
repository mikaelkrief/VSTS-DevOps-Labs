## Docker Deployment using VSTS and Azure for ASP.NETCORE application

## Overview

This lab shows how to build custom images of <a href="https://docs.docker.com/engine/examples/dotnetcore/">**Dockerized ASP.NETCORE**</a> application, push those images to <a href="https://docs.docker.com/registry/"> **Private Repository** </a> (<a href="https://azure.microsoft.com/en-in/services/container-registry/"> Azure Container Registry </a>), and pull these images to deploy to containers in **Azure Web App** (Linux) using Visual Studio Team Services.

Web App for containers lets you bring your own <a href="https://www.docker.com/what-docker">Docker</a> formatted container images, easily deploy and run them at scale with Azure. Combination of Team Services and Azure integration with Docker will enable you to:

1.  <a href="https://docs.docker.com/engine/reference/commandline/build/"> Build </a> your own custom images using <a href="https://docs.microsoft.com/en-us/vsts/build-release/concepts/agents/hosted"> VSTS Hosted Linux agent </a>
2. <a href="https://docs.docker.com/engine/reference/commandline/push/"> Push </a> and store images in your private repository
3. Deploy and  <a href="https://docs.docker.com/engine/reference/commandline/run/"> run </a> images inside containers

Below screenshot helps you understand the VSTS DevOps workflow with Docker: 

<img src="images/vstsdockerdevops.png">


## Pre-requisites

1.  **Microsoft Azure Account**: You need a valid and active azure account for this lab.

2. You need a **Visual Studio Team Services Account** and <a href="https://docs.microsoft.com/en-us/vsts/accounts/use-personal-access-tokens-to-authenticate">Personal Access Token</a>.

3. You need to install **Docker Integration** extension from <a href="https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.docker">Visual Studio Marketplace</a>.

## Setting up the Environment

We will create an **Azure Container Registry** (ACR) to store the images generated during VSTS build. These images contain environment configuration details with build settings.  An **Azure Web App** (with Linux OS) is created where custom built images will be deployed to run inside container (single container). **Azure SQL Database** along with **SQL Server** is created as a backend to **MyHealthClinic** .NetCore sample application.

1. Click on **Deploy to Azure** (or right click and select ***Open in new tab***) to spin up **Azure Container Registry**, **Azure Web App** and **Azure SQL Database** along with **Azure SQL Server**. Enter required details such as Acr name, Site Name and DB Server Name. Agree to ***Terms and Conditions***, and click **Purchase**.

   <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMicrosoft%2FVSTS-DevOps-Labs%2Fdocker%2Fdocker%2Ftemplates%2Ftemplate.json" target="_blank">
   <img src="http://azuredeploy.net/deploybutton.png"/>
   </a> 
   <br/>
      
      Click  <a href="https://azure.microsoft.com/en-in/regions/services/"> here </a> to to see Azure products available by region.

    >**Note**: Use small case letters for ***DB Server Name***.

   <br/>
   <img src="images/createazurecomponents.png">

2. It takes approximately **3 to 4 minutes** to provision the environment. Click on **Go To resource group**.

   <img src="images/deploymentsucceeded.png">

3. Below components are created post deployment.
     
    <table width="100%">
     <thead>
      <tr>
         <th width="50%"><b>Azure Components</b></th>
         <th><b>Description</b></th>
      </tr>
    </thead>
    <tr>
      <td><img src="images/container_registry.png" width="30px"><b>Azure Container Registry</b></td>
      <td>Used to store images privately</td>
    </tr>
    <tr>
      <td><img src="images/storage.png" width="30px"> <b>Storage Account</b></td>
      <td>Container Registry resides in this storage account</td>
    </tr>
    <tr>
      <td><img src="images/app_service.png" width="30px"> <b>App Service</b></td>
      <td>Docker images are deployed to containers in this App Service</td>
    </tr>
    <tr>
      <td><img src="images/app_service_plan.png" width="30px"> <b>App Service Plan</b></td>
      <td>Resource where App Service resides</td>
    </tr>
    <tr>
      <td><img src="images/sqlserver.png" width="30px"> <b>SQL Server</b> </td>
      <td>SQL Server to host database</td>
    </tr>
    <tr>
      <td><img src="images/sqldb.png" width="30px"> <b>SQL database</b> </td>
      <td>SQL database to host MyHealthClinic data</td>
    </tr>
    </table>

    </br>

   <img src="images/postazuredeployment.png">
   

4. Click on **mhcdb** SQL database. Note down the **Server name**.

   <img src="images/getdbserverurl.png">

5. Go back to your resource group. Click on container registry and note down the **Login server** name. We need these details later in Exercise 2.

   <img src="images/getacrserver.png">
   

## Setting up the Project

1. Use <a href="https://vstsdemogenerator.azurewebsites.net/?name=Docker&templateid=77363" target="_blank">VSTS Demo Data Generator</a> to provision a project on your VSTS account 

    <img src="images/VSTSDemogenerator.png">

2. Provide a Project Name, and click on Create Project.

   <img src="images/vstsdemogen2.png">

3. Once the project is provisioned, click the **URL** to navigate to the project.

   <img src="images/vstsdemogen3.png">


## Exercise 1: Endpoint Creation

Since the connections are not established during project provisioning, we will manually create the Azure endpoint. 

1. In VSTS, navigate to **Services** by clicking on the gear icon <img src="images/gear.png">, and click on **+ New Service Endpoint**. Select **Azure Resource Manager**. Specify **Connection name**, select your **Subscription** from the dropdown and click **OK**. We use this endpoint to connect **VSTS** and **Azure**.

   <img src="images/azureendpoint.png">

   You will be prompted to authorize this connection with Azure credentials. Disable pop-up blocker in your browser if you see a blank screen after clicking OK, and retry the step. 


## Exercise 2: Configure CI-CD

 Now that the connection is established, we will manually map the Azure endpoint and Azure Container Registry to build and release definitions. We will also deploy the dacpac to mhcdb database so that the schema and data is set for the backend.
 
>Note : If you encounter an error - ***TFS.WebApi.Exception: Page not found*** for Azure tasks in the build/ release definition, you can fix this by typing a random text in the Azure Subscription field and click the **Refresh** icon next to it. Once the field is refreshed, you can select the endpoint from the drop down. This is due to a recent change in the VSTS Release Management API. We are working on updating VSTS Demo Generator to resolve this issue.

1. Go to **Builds** under **Build and Release** tab, **Edit** the build definition **Docker**.

   <img src="images/build.png">

2. In the **Process** section, update **Azure subscription** and **Azure Container Registry** with the endpoint component from the dropdown. (use arrow keys to choose Azure Container Registry for the first time). Click **Save**.

   <img src="images/updateprocessbd.png">

   <br/>
   <br/>

   <table width="100%">
   <thead>
      <tr>
         <th width="50%"><b>Tasks</b></th>
         <th><b>Usage</b></th>
      </tr>
   </thead>
   <tr>
      <td><img src="images/icon.png"><b>Run services</b></td>
      <td>prepares suitable environment by restoring required packages</td>
   </tr>
   <tr>
      <td><img src="images/icon.png"><b>Build services</b></td>
      <td>builds images specified in a <b>docker-compose.yml</b> file with registry-qualified names and additional tags such as <b>$(Build.BuildId)</b></td>
   </tr>
    <tr>
      <td><img src="images/icon.png"><b>Push services</b></td>
      <td>pushes images specified in a <b>docker-compose.yml</b> file, with multiple tags, to container registry</td>
   </tr>
    <tr>
      <td><img src="images/icon.png"><b>Lock services</b></td>
      <td>pulls image from default tag <b>latest</b> in container registry and verifies if uploaded image is up to date</td>
   </tr>
   <tr>
      <td><img src="images/copy-files.png"><b>Copy Files</b></td>
      <td>used to copy files from source to destination folder using match patterns </td>
   </tr>
   <tr>
      <td><img src="images/publish-build-artifacts.png"><b>Publish Build Artifacts</b> </td>
      <td> used to share the build artifacts </td>
   </tr>
   </table>

3. Go to **Releases** under **Build & Release** tab, **Edit** the release definition **Docker** and select **Tasks**.

   <img src="images/release.png">
   <br/>
   <br/>
   <img src="images/release_tasks.png">

4. Description of three phases used in this release are given below:

   <table width="100%">
   <thead>
      <tr>
         <th width="50%"><b>Phases</b></th>
         <th><b>Usage</b></th>
      </tr>
   </thead>
   <tr>
      <td><b>DB deployment</b></td>
      <td><b>Hosted VS2017</b>  agent is used to create database schema along with pre-configured data in <b>mhcdb</b></td>
   </tr>
   <tr>
      <td><b>Agentless phase</b></td>
      <td><b>Manual intervention</b> used to confirm if Azure Container Registry is manually mapped with Azure Web App</td>
   </tr>
   <tr>
      <td><b>Agent phase</b></td>
      <td><b>Hosted Linux Preview</b> agent is used to pull image from ACR and deploy in Linux Web App</td>
   </tr>
   </table>

5. Under **Execute Azure SQL:DacpacTask**, update **Azure Subscription** from the dropdown. 

    **Execute Azure SQL:DacpacTask** will deploy the dacpac to **mhcdb** database so that the schema and data is set for the backend.

    <img src="images/update_dbtask.png">

6. Under **Azure App Service Deploy** task, update **Azure subscription** and **Azure Service name** with the endpoint components from the dropdown.

    **Azure App Service Deploy** will pull the appropriate image corresponding to the BuildID from repository specified, and deploys the image to Linux App Service. **Manual Intervention** step is used to confirm if Azure Container Registry is mapped with Azure Web App.

    <img src="images/updatedrd.png">

7. Click on **Variables** section, update **ACR** and **SQLserver** with the details noted earlier while setting up the environment. Click **Save**. 

    <img src="images/update_rdvariables.png">

    >Note: **Database Name** is set to **mhcdb**, **Server Admin Login** is **sqladmin** and **Password** is **P2ssw0rd1234**.


## Exercise 3: Trigger CI-CD with Code Change

In this exercise, we will update the code to trigger CI-CD. 

1. Go to **Files** under **Code** tab, and navigate to the below path to **Edit** the file **Index.cshtml** 

   >Docker/src/MyHealth.Web/Views/Home/**Index.cshtml**

   <img src="images/editcode.png">

2. Go to line number **28**, update **JOIN US** to **JOIN US TODAY**, and click **Commit**.

    <img src="images/lineedit.png">

3. Click **Commit** in the pop-up window.

    <img src="images/commit.png">

4. Go to **Builds** tab. Click on the build number to see the build in progress.

    <img src="images/build3.png">

    Build will generate and push the image to Azure Container Registry. After build completes, you will see the build summary. 
    
    <img src="images/build4.png">

5. Go to <a href="https://portal.azure.com">Azure Portal</a>, navigate to the **App Service** which was created at the beginning of this lab. Select **Docker Container** section. Under **Image Source** highlight **Azure Container Registry**. Select your **Registry** from the dropdown. Under **image** dropdown select **myhealth.web** and under **Tag** dropdown select **latest**. This is required to map Azure Container Registry with the Web App. Click **Save**.

    <img src="images/updatereg.png">
    <br/>
    <br/>
    <img src="images/updatereg2.png">

6. To see generated images, go to your **Azure Container Registry** and navigate to **Repositories**.

    <img src="images/imagesinrepo.png">

7. Switch back to **Releases** in VSTS, and double click on latest release. 

    <img src="images/rel0.png">

8. Navigate to **Logs** section to see the release in progress. It takes upto 4 to 5 minutes for dacpac deployment task to complete.

    <img src="images/rel3.png">

9. After dacpac deployment task is complete, click on **Resume or Reject**.

    <img src="images/rel5.png">

10. If you have already mapped the Azure Container Registry with the Azure Web App, give some comment and Click **Resume**. If not, go back to step 5 and then come back to this step.

    <img src="images/rel6.png">

11. The release will deploy the image to App Service based on the **BuildID**, which is tagged with the image. You will see below summary once the release is complete.
    
    <img src="images/rel8.png">

12. Switch back to <a href="https://portal.azure.com">Azure Portal</a>, navigate to the **Overview** section of your **App Service**. Click on the **URL** to see the changes in your app.

    <img src="images/getwebappurl.png">
    <br/>
    <br/>
    <img src="images/finalresult.png">

    Use below credentials to login to your **HealthClinic** app:

    **Username**: user

    **Password**: P2ssw0rd@1
    

## Summary

With **Visual Studio Team Services** and **Azure**, we can build DevOps for dockerized applications by leveraging docker capabilities enabled on VSTS Hosted Agents.

## Feedback

Please let <a href="mailto:devopsdemos@microsoft.com">us </a> know if you have any feedback on this lab.