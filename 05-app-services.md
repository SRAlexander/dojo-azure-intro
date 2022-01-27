# App Services

An easy way to host an API or Frontend application without having to configure a VM! 
First off lets head to the Azure Portal...

1. Click "Create a resource" 
2. Search for Web App
3. Create!

Basics Page

Subscription - default
Resource group - create a new one or add it to an existing group
Name - Your selection
Publish - Code
Runtime Stack - Node 14 LTS
Operating System - Linux 
Region - Uk South
SKU and Size - Make sure you select a Dev instance for now, B1


Following that we will just create the application. Have a quick once over to see what will be created on the Review + Create page before clicking Create.

![alt](images/azure-cweb-app-create-page.png)

Once the deployment is complete have a look at the finalising page. You should be able to see a deployment details section, if you ever plan to use the CLI for IoC then you can download a json file of the deployment details here to preproduce later. 

Lets click on "Go to resource" and you should see the following page...

![alt](images/azure-web-app-home.png)

First things first, click the browse button or the url in the form of... https://<name>.azurewebsites.net

![alt](images/azure-web-app-default-web-page)

Nice, we have a functional web app. Once key thing to notice is that https:// is configured by default whcih can be a pain to create, manage and update on a standard VM.


Deployment Center

Now lets click on the deployment center and host something of our own. 
First of all, since we have set the runtime to node, we can run an angular application quite easily from this service. For the purpose of this demo I've create a default angular application here... https://github.com/SRAlexander/test-ng-12, grab a copy and host it on your Github account.

Back to the Azure Portal...

1. Select our Source as GitHub
2. Sign into your github account
3. Select the orignasation as yourself
4. Select the repository we've just created.
5. You can even select the specific branch, main is fine for now. 
6. Click preview file, this is going to create a Github workflow file against our GitHub repository
7. Now click Save

If we quickly shoot over to our github repository and click on the actions tab we can see a build in progress... 

![alt](images/azure-web-app-github-build-first)

However we are going to stop this build, click on the build and then click cancel workflow. We are then going to make some specific changes the workflow file. 

1. click on the three dots to the top left
2. selet view workflow file
3. click the edit button
4. replace with the following, making sure you update the secrets.AZUREAPPSERVICE_PUBLISHPROFILE_xxxxxxx with your instance...

    name: Build and deploy Node.js app to Azure Web App - grad-demo

    on:
    push:
        branches:
        - main
    workflow_dispatch:

    jobs:
    build:
        runs-on: ubuntu-latest

        steps:
        - uses: actions/checkout@v2

        - name: Set up Node.js version
            uses: actions/setup-node@v1
            with:
            node-version: '14.x'

        - name: npm install, build, and test
            run: |
            npm install
            npm run build --prod --if-present
            working-directory: .

        - name: 'Deploy to Azure Web App'
            id: deploy-to-webapp
            uses: azure/webapps-deploy@v2
            with:
            app-name: 'grad-demo'
            slot-name: 'Production'
            publish-profile: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_XXXXXXXXXX }}
            package: ./dist/angular12-dojo

The key thing to note here is that we are not going to run the test process as that requires chrome for Angular and we are only deploying our dist folder as we don't need anything else.

Commit the file to your main branch and revisit the actions tab, our new build will have kicked off. Click into it to watch what it's doing if you like.
Once complete head back over to Azure we have one final configuration to make.

   1. head to the configuration page on the app
   2. click on the "general settings tab"
   3. place the following into the Startup command... "pm2 serve /home/site/wwwroot --no-daemon --spa"

This tells the app to serve static files from the wwwroot folder which is where our application is currently sitting. The --spa tab means all url requests will be redirected through the index.html. This is impootant to take advanatge of Angular routing.

Have a look at our hosted page again and we should now see our Angular demo app up and running. Nice job!


