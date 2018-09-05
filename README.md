# aws-fargate-jupyterhub-demo
Details the steps to deploy a docker container running JupyterHub to AWS ECS (Fargate)

To test the docker container starts JupyterHub successfully, run
* First build the docker image `$ docker build -t jupyterhub/test .`
* Then run the container `docker run --rm -ti -p 8000:8000 jupyterhub/test`
* From your browser navigate to `http://localhost:8000` and verify you can see the JupyterHub home page

Assuming everything went well in step 1, let's create a new docker registry in Amazon Elastic Container Registry (ECR) using the AWS CLI and push our container image to it. Fargate will pull our container from ECR to run. 

* From your command line/terminal run `$ aws ecr create-repository --repository-name jupyterhub/test`. Take note of the `repositoryUri` that is returned
* Authenticate our docker client to the ECR registory: `$(aws ecr get-login --no-include-email --region us-east-1)`
* Tag the image: `docker tag jupyterhub/test:latest <the_repositoryUri_returned_earlier>:latest`
* And lastly, push the image: `docker push <the_repositoryUri_returned_earlier>:latest`
* Lets verify that our image made it into ECR: `aws ecr list-images --repository-name jupyterhub/test`


