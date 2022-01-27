# App Services with ACR

Deploying ana pp service through a build proces sis great, but every time we want to deploy it to an isntance we have to rebuild it! We can use a contonarisation to create an image of our application to deploy it instantly after an initial build! So for the first step, lets create an Azure Container Registry.

1. Search for Container Registry
2. Select the option
3. Click Create
4. Add the following details
   1. Resource group = the same one you created/used for section 5
   2. Registry name = give it a name 
   3. Location = Uk South
   4. SKU = Basic
5. We wil leave everything else alone, continue to the Review + Create page
6. Click Create
7. Once created head into the the resource and head to the the "Access Keys section"
8. For the purposes of this demo enable the admin user

![alt](images/azure-acr-settings.png)

Next up we are going to need to build a Docker Image, fortunatly I've already done this for you but lets take a look at what it consists of...

    FROM node:14
	# stage 1

	FROM node:alpine AS my-app-build
	WORKDIR /app
	COPY . .
	# RUN BUILDENV
	RUN npm install && npm run build --prod

	# stage 2

	FROM nginx:alpine
	COPY --from=my-app-build /app/dist/angular12-dojo /usr/share/nginx/html
	COPY --from=my-app-build /app/nginx.conf /etc/nginx/conf.d/default.conf
	EXPOSE 80

Stage 1 is our application build step, we are using a default node image as a starting point and building our application
Stage 2 is our hosting section, nginx is a local server which requires our build files, a server config file and an access port, 80 in this case.

The third step is to create a new Github Action to build our Docker image and push it into the ACR

    env:
        "IMAGE_TAG": "1.0"
        "IMAGE_NAME_FULL" : "project_grad_demo"
        "ENV_NAME" : "dev"

    push-to-systest-acr:
        runs-on: ubuntu-latest
        steps:
        - name: Checkout
            uses: actions/checkout@v2

        - name: Login to Azure
            uses: Azure/login@v1
            with:
            creds: ${{ secrets.AZURE_CREDENTIALS }}
            
        - name: Login to ACR
            uses: azure/docker-login@v1
            with:
            login-server: ${{ secrets.REGISTRY_LOGIN_SERVER }}
            username: ${{ secrets.AZURE_CLIENT_ID }}
            password: ${{ secrets.AZURE_CLIENT_SECRET }}
                
        - name: Build Docker Image
            run: docker build -t ${{ secrets.REGISTRY_LOGIN_SERVER }}/${{ env.IMAGE_NAME_FULL }}:${{ env.IMAGE_TAG }} --build-arg BUILDENV=${{env.ENV_NAME}} .      
            
        - name: Push the image to ACR
            run: docker push ${{ secrets.REGISTRY_LOGIN_SERVER }}/${{ env.IMAGE_NAME_FULL }}:${{ env.IMAGE_TAG }}
    
        - name: Invoke deployment hook
            uses: distributhor/workflow-webhook@v1
            env:
            webhook_type: 'json-extended'
            webhook_url: ${{ secrets.WEBHOOK_URL }}
            webhook_secret: ${{ secrets.WEBHOOK_SECRET }} 


So the steps should be self explanatory, however, this will not work due to missing variables. You can see I have references to secrets throughout the script where confidential or sensitive data items are required. We will be taking advantage of Githubs sensitive data implementation to store these variables but first we need to create them. 

Lets create a Service Principal, in the Azure CLI run the following command using your Subscription ID, 

az ad sp create-for-rbac -n "GIVE ME A NAME" --role Contributor --scopes /subscriptions/<Your Subscription Id>>/resourceGroups/<Your Resource Group Name> --sdk-auth

If the command is successfull then you will get a json object in the following form...

    {
        "clientId": "xxxxxx-xxxx-xxx-xxxx",
        "clientSecret": "xxxxxx-xxxx-xxx-xxxx",
        "subscriptionId": "xxxxxx-xxxx-xxx-xxxx",
        "tenantId": "xxxxxx-xxxx-xxx-xxxx",
        "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
        "resourceManagerEndpointUrl": "https://management.azure.com/",
        "activeDirectoryGraphResourceId": "https://graph.windows.net/",
        "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
        "galleryEndpointUrl": "https://gallery.azure.com/",
        "managementEndpointUrl": "https://management.core.windows.net/"
    }

I'm afraid to say that once an app service has been created with a deployment option, it cannot be changed, therefore we need to create a second app service like in task 5 but this time we need to set the deployment option to "container". Give it a go.

Once up and running, we will have access to all the information we need to set up our secrets. github > repo > settings > secrets

AZURE_CREDENTIALS = is the above model
AZURE_CLINET_ID = extract the client id from above
AZURE_CLIENT_SECRET = extract the client secret from above
REGISTRY_LOGIN_SERVER = found on the ACR overview page
WEBHOOK_URL = found on the app service deployment center page, even though we cannot set our image yet, we can get the url by displaying the webhook

Trigger off the new build by commiting the new changes to the repository from above. If we now look in the actions tab on github we should see both of our deployments taking place. Once again you can watch the new process but as soon as it's green we can finally link up our new app service with the new container image. 

Head to the app service and open the deployment center, we can now set the following options..

   1. Container Type = Single Container
   2. Registry Source = Azure Container Registry
   3. Subscription ID as is
   4. Registry = Your registry name
   5. image = project_grad_demo
   6. tag = 1.0
   7. Continuous Deployment = true

Then click save. It will take a few moments for the configuration to take place but once complete, visit the browser page for the app. If all has been successful you will have now have hosted your first Docker container as an Azure Web App. Congrats :) 