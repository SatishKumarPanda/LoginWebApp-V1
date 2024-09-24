pipeline {
    agent any 
    tools {
        maven 'Maven'
    }

    environment {
        AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
    }      

    stages {
        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'mvn clean install'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
        }
        
        stage('Publish') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn package'
            }
        }

        stage('Upload to S3') {
            steps {
                echo 'Configuring AWS and uploading artifact...'
                sh 'aws configure set region ap-south-1'
                sh 'aws s3 cp ./target/*.war s3://jenkinsucket01'
            }
            post {
                success {
                    echo 'Archiving artifacts...'
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        }

        stage('Prepare Docker Directory on Server') {
            steps {
                sshagent(['Tomcat']) {
                    echo 'Creating directory for Dockerfiles on remote server...'
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.111.169.66 "
                        mkdir -p /home/ec2-user/dockerfiles
                        "
                    '''
                }
            }
        }

        stage('Copy Dockerfiles to Server') {
            steps {
                sshagent(['Tomcat']) {
                    echo 'Copying Dockerfiles to remote server...'
                    sh '''
                        scp -o StrictHostKeyChecking=no  Dockerfile-mysql ec2-user@3.111.169.66:/home/ec2-user/dockerfiles/
                        scp -o StrictHostKeyChecking=no Dockerfile-tomcat ec2-user@3.111.169.66:/home/ec2-user/dockerfiles/
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sshagent(['Tomcat']) {
                    echo 'Building Docker images on remote server...'
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.111.169.66 "
                        cd /home/ec2-user/dockerfiles &&
                        docker build -t my-image-1 -f Dockerfile-mysql . &&
                        docker build -t my-image-2 -f Dockerfile-tomcat .
                        "
                    '''
                }
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(['Tomcat']) {
                    echo 'Running Docker containers on remote server...'
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.111.169.66 "
                        docker run -d --name my-mysql-container -p 8081:8080 my-image-1 &&
                        docker run -d --name my-tomcat-container -p 8082:8080 my-image-2
                        "
                    '''
                }
            }
        }
    }
}
