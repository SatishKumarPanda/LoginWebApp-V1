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

        stage('Copy Dockerfiles to Server') {
            steps {
                sshagent(['webserver']) {
                    echo 'Copying Dockerfiles to Docker server...'
                    sh '''
                        scp -o StrictHostKeyChecking=no Dockerfile-mysql ec2-user@65.0.89.79:/path/to/dockerfiles/
                        scp -o StrictHostKeyChecking=no Dockerfile-tomcat ec2-user@65.0.89.79:/path/to/dockerfiles/
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
                        cd /path/to/dockerfiles &&
                        docker build -t my-mysql-image -f Dockerfile-mysql . &&
                        docker build -t my-tomcat-image -f Dockerfile-tomcat .
                        "
                    '''
                }
            }
        }

        stage('Run Docker Containers') {
            steps {
                sshagent(['Tomcat']) {
                    echo 'Running Docker containers on remote server...'
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.111.169.66"
                        docker run -d --name mysql-container -e MYSQL_ROOT_PASSWORD=my-secret-pw -p 3306:3306 my-mysql-image &&
                        docker run -d --name tomcat-container -p 8081:8080 my-tomcat-image
                        "
                    '''
                }
            }
        }
    }
}
