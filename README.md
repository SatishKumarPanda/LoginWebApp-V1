# Jenkins Pipeline for Building and Deploying Applications

## Overview

This Jenkins pipeline automates the process of building a Java application, running tests, packaging the application, uploading artifacts to AWS S3, and deploying Docker containers on a remote server. The pipeline also includes cleanup stages to ensure that temporary files and unused Docker images are removed from both the Jenkins master server and the deployment server.

## Prerequisites

- Jenkins installed and configured
- Jenkins plugins: **Pipeline**, **SSH Agent**, **AWS Steps**
- Maven installed on Jenkins
- Docker installed on the deployment server
- AWS CLI configured on Jenkins
- SSH access to the deployment server
- Necessary credentials stored in Jenkins for AWS and Docker Hub

## Environment Variables

- `AWS_ACCESS_KEY_ID`: AWS access key ID (configured in Jenkins credentials)
- `AWS_SECRET_ACCESS_KEY`: AWS secret access key (configured in Jenkins credentials)
- `DOCKERHUB_CREDENTIALS`: Docker Hub credentials (username and password)

## Pipeline Stages

1. **Build**
   - Compiles the application using Maven.
   - Command: `mvn clean install`

2. **Test**
   - Runs unit tests using Maven.
   - Command: `mvn test`

3. **Publish**
   - Packages the application into a WAR file.
   - Command: `mvn package`

4. **Upload to S3**
   - Configures AWS CLI to set the region.
   - Uploads the generated WAR file to an S3 bucket.
   - Command: `aws s3 cp ./target/*.war s3://jenkinsucket01`

5. **Prepare Docker Directory on Server**
   - Creates a directory on the remote server for Dockerfiles and artifacts.
   - Command: `mkdir -p /home/ec2-user/dockerfiles`

6. **Copy Dockerfiles to Server**
   - Copies necessary Dockerfiles and the WAR file to the remote server.
   - Uses SCP to transfer files:
     ```bash
     scp -o StrictHostKeyChecking=no Dockerfile-mysql ec2-user@3.111.169.66:/home/ec2-user/dockerfiles/
     ```

7. **Build Docker Images**
   - Builds Docker images using the copied Dockerfiles on the remote server.
   - Command:
     ```bash
     docker build -t my-image-1 -f Dockerfile-mysql . &&
     docker build -t my-image-2 -f Dockerfile-tomcat .
     ```

8. **Login to Docker Hub**
   - Logs into Docker Hub to prepare for pushing images.
   - Command:
     ```bash
     echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
     ```

9. **Push to Docker Hub**
   - Pushes the built Docker images to Docker Hub.
   - Command:
     ```bash
     docker push my-image-1 &&
     docker push my-image-2
     ```

10. **Deploy to Server**
    - Runs the Docker containers on the remote server.
    - Command:
      ```bash
      docker run -d --name my-mysql-container -p 8081:8080 my-image-1 &&
      docker run -d --name my-tomcat-container -p 8082:8080 my-image-2
      ```

11. **Clean Master Server**
    - Cleans up temporary files created during the build process on the Jenkins master server.
    - Command:
      ```bash
      rm -rf Dockerfile-mysql Dockerfile-tomcat dump target/LoginWebApp.war
      ```

12. **Clean Deployment Server**
    - Cleans up Docker images and temporary files on the deployment server.
    - Command:
      ```bash
      docker rmi my-image-1 my-image-2 &&
      rm -rf /home/ec2-user/dockerfiles/*
      ```

## Usage

1. Set up your Jenkins server with the necessary plugins and tools.
2. Create a new pipeline job in Jenkins.
3. Paste the provided pipeline code into the job configuration.
4. Ensure that all necessary credentials are configured in Jenkins.
5. Run the pipeline to build, test, package, and deploy your application.

## Notes

- Make sure to replace any hardcoded values (like server IPs or bucket names) with your own configurations as necessary.
- Monitor the Jenkins console output for logs during each stage for troubleshooting.

## Conclusion

This Jenkins pipeline provides a complete solution for automating the build and deployment process of a Java application using Maven and Docker. Follow the steps outlined above to set up and use this pipeline effectively.
