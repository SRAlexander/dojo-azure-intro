# Infrastructure as code and cognitive services

As an introduction to infrastructure as code and cognitive services we will:

1. Create a simple Bicep template and use it to deploy an instance of Cognitive services.
2. Run a simple program from the VM to transcribe audio using your Cognitive Services instance.

## Deploying cognitive services

With Infrastructure as Code (IaC) we declaratively define the Azure resources we want, and then deploy this template to create the resources. 
Template deployment is idempotent meaning repeated deployments will have no effect: they simply confirm the resources have been created.