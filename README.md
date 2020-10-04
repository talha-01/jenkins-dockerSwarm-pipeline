# Create Docker Swarm Infrastructure and Deploy Flask Application
This project creates a Docker Swarm cluster and deployes an application in this order:

- Creates a repo in the ECR.

- Builds the application image.

- Creates an application container, a database container, and a the network to test the image.

- Pushes the image to the ECR repo.

- Creates the Docker Swarm cluster using the Cloud Formation template, which can be found in this repository.

- Tests if the cluster and the Docker Service is running.

- After Docker service starts running, deploys the application.

- Tests if the application is running.


The Jenkins Server is using an Amazon Linux 2 AMI. It also has Docker, and Docker, Docker Commons, Docker Pipeline, SSH Agent, SSH Pipeline Steps plugins installed.
The Jenkins Server should have a role with,

- AmazonEC2FullAccess

- AmazonEC2ContainerRegistryFullAccess

- AWSCloudFormationFullAccess

- IAM managed policy (can be found in the repository in JSON format)


Jenkinsfile is getting the account number using the the Jenkins Server's name tag. In order to use this feature, you can tag your Jenkins Server as `JenkinsServer`, or you can enter your account number directly.

