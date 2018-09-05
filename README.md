# aws-fargate-jupyterhub-demo
Details the steps to deploy a docker container running JupyterHub to AWS ECS (Fargate). In this guide we are going to 
1. Build the JupyterHub docker image on our local computer and run it to verifying it is working
2. Create an Amazon Elastic Container Registry (Amazon ECR) repository to hold the JupyterHub container
3. Create an Amazon Elastic Container Service (Amazon ECS) Cluster to hold the running JupyterHub container
4. Create a Task Definition to specify CPU/Memory/Port configuration for the JupyterHub container
5. Create a Service that will run the Task Definition on the Amazon ECS Cluster

# Build the Docker Image

To test the docker container starts JupyterHub successfully, run
* First build the docker image `$ docker build -t jupyterhub/test .`
* Then run the container `docker run --rm -ti -p 8000:8000 jupyterhub/test`
* From your browser navigate to `http://localhost:8000` and verify you can see the JupyterHub home page

Assuming everything went well in step 1, let's create a new docker registry in Amazon Elastic Container Registry (ECR) using the AWS CLI and push our container image to it. Fargate will pull our container from ECR to run. 

# Create the ECR Repository

* From your command line/terminal run `$ aws ecr create-repository --repository-name jupyterhub/test`. Take note of the `repositoryUri` that is returned
* Authenticate our docker client to the ECR registory: `$(aws ecr get-login --no-include-email --region us-east-1)`
* Tag the image: `docker tag jupyterhub/test:latest <the_repositoryUri_returned_earlier>:latest`
* And lastly, push the image: `docker push <the_repositoryUri_returned_earlier>:latest`
* Lets verify that our image made it into ECR: `aws ecr list-images --repository-name jupyterhub/test`

# Create the ECS Cluster

* Create a new VPC only cluster from the AWS ECS Console, choose the `Networking Only` template
* Give it a name, e.g., `jupyterhub-cluster` and select Create a new VPC to house it.

# Create the Task Definition

* Create a new `Task Definition`. Choose Fargate, give it a name. The `Task Role` is important if the JupyterHub task needs to access other services. For Task size, choose 2GB memory and 1 vCPU. 
* Next choose `Add Container`, give it a name, and set the Image field to the `<the_repositoryUri_returned_earlier>`. Set the soft limit to 512 and specify `8000` for the port mapping. Then click create. 

# Create the Service

* Now we need to create a `Service` (ECS speak for a long-running task). From the Task Definitions link on the left, select the new task definition created in the previous step. Then select Actions, Create Service. Choose Fargate for the Launch Type, specify the cluster you created earlier and give this new service a name. Leave service type as replica, and specify 1 for number of tasks. Leave the rest of the fields alone and click Next Step. 
* Choose the VPC associated with the cluster and go ahead and a subnet to deploy the service in to. Click Edit next to the Security Groups and add a TCP rule allowing inbound traffic on port 8000. Uncheck Enable service discovery integration and choose Next Step. Choose Next Step again and then Create.

* Once the Service is finished being created we can view it. From the service page, choose Tasks, and then select the Task in the table below. Monitor the status of the deployment until it says RUNNING.

** Note that Amazon ECS needs to be able to pull the ECR image over the internet. See below for details: 
```
If you are launching a task without a public IP, make sure that the route table on the subnet has "0.0.0.0/0" going to a NAT Gateway or NAT instance to ensure access to the internet. If your route table has an internet gateway, this is acting like a firewall and preventing the connection from being made. If you are launching a task with a public IP, make sure that the route table on the subnet has "0.0.0.0/0" going to an internet gateway to ensure you will be able to use the public IP successfully for ingress traffic.
Verify your security group rules for the Task allows for outbound access. The default here is typically All Traffic to 0.0.0.0/0.
```
