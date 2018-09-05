In this guide we are going to run a JupyterHub container on Amazon Elastic Container Service (Fargate) by following the steps below:

1. Build a JupyterHub docker image on our local computer and run it to verifying it works
2. Create an Amazon Elastic Container Registry (Amazon ECR) repository to hold the JupyterHub container
3. Create an Amazon Elastic Container Service (Amazon ECS) Cluster to hold the running JupyterHub container
4. Create a Task Definition to specify CPU/Memory/Port configuration for the JupyterHub container
5. Create a Service that will run the Task Definition on the Amazon ECS Cluster

This guide assumes you have the AWS CLI installed and have run `$ aws configure` with appropriate IAM priveleges to interact with Amazon ECR and Amazon ECS.

# Build the Docker Image

To test the docker container starts JupyterHub successfully we will build the container locally and verify we can navigate to the webpage being hosted from it.

* First build the docker image `$ docker build -t jupyterhub/test .`
* Then run the container `docker run --rm -ti -p 8000:8000 jupyterhub/test`
* From your browser navigate to `http://localhost:8000` and verify you can see the JupyterHub home page

# Create the Aamzon ECR Repository

Amazon Elastic Container Registry is a fully-managed, private Docker container registry. We will use it to hold our built JupyterHub container.

* From your command line/terminal run `$ aws ecr create-repository --repository-name jupyterhub/test`. Take note of the `repositoryUri` that is returned
* Authenticate our docker client to the ECR registory: `$(aws ecr get-login --no-include-email --region us-east-1)`
* Tag the image: `docker tag jupyterhub/test:latest <the_repositoryUri_returned_earlier>:latest`
* And lastly, push the image: `docker push <the_repositoryUri_returned_earlier>:latest`
* Lets verify that our image made it into ECR: `aws ecr list-images --repository-name jupyterhub/test`

# Create the Amazon ECS Cluster

Amazon Elastic Container Service is a orchestration service for Docker containers. It integrates with Amazon ECR and can pull container images hosted there.

* Create a new VPC only cluster from the AWS ECS Console, choose the `Networking Only` template
* Give it a name, e.g., `jupyterhub-cluster` and select Create a new VPC to house it.

# Create the Task Definition

A Task Definition describes the parameters of your Docker container: what image to use, cpu/memory constraints, etc...

* Create a new `Task Definition`. Choose Fargate, give it a name. The `Task Role` is important if the JupyterHub task needs to access other services. For Task size, choose 2GB memory and 1 vCPU. 
* Next choose `Add Container`, give it a name, and set the Image field to the `<the_repositoryUri_returned_earlier>`. Set the soft limit to 512 and specify `8000` for the port mapping. Then click create. 

# Create the Service

In Amazon ECS you can create short-lived tasks (think cron jobs) or long-running tasks (think 24x7 web service). In this case we will want to create a service, and we we use the Task Definition from the previous step to start. 

* From the Task Definitions link on the left, select the new task definition created in the previous step. 
* Then select Actions, Create Service. Choose Fargate for the Launch Type
* Specify the cluster you created earlier and give this new service a name. Leave service type as replica, and specify 1 for number of tasks. Leave the rest of the fields alone and click Next Step. 
* Choose the VPC associated with the cluster and go ahead and a subnet to deploy the service in to. Click Edit next to the Security Groups and add a TCP rule allowing inbound traffic on port 8000 (the port that JupyterHub listens on). Uncheck Enable service discovery integration and choose Next Step. Choose Next Step again and then Create.

* Once the Service is finished being created we can view it. From the service page, choose Tasks, and then select the Task in the table below. Monitor the status of the deployment until it says RUNNING. The `Public IP` under the `Network` section shows the public IP address. From you internet browser navigate to <the_public_ip>:8000 and you should see JupyterHub home page.


