pipeline {
    agent {
        label 'master'
    }

    environment {
        PROJECT="project-204"
        ACCOUNT_ID=sh(script: '''aws ec2 describe-instances --region us-east-1 --filter "Name=tag:Name,Values='JenkinsServer'" --query=Reservations[*].[OwnerId] --output text''', returnStdout: true).trim()
        APP_NAME="${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}"
        REPO_VERSION="App-${BUILD_NUMBER}"
        DOCKERFILE_URL="https://github.com/talha-01/jenkins-dockerSwarm-pipeline.git"
        GIT_FILE_URL="https://raw.githubusercontent.com/talha-01/jenkins-dockerSwarm-pipeline/master/"
    }


    stages {

        stage('CREATE REPO'){
            steps{
                echo 'Creating the ECR repo'
                sh '''
                aws ecr create-repository \
                  --repository-name ${PROJECT} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region us-east-1 || echo 'Repo already exists'
                '''                
            }
        }
        stage('CONTROL REPO') {
            steps{
                echo 'Checking the ECR repo'
                sh 'aws ecr describe-repositories --region us-east-1 | grep $PROJECT && echo "Using existing repo"'
            }
        }
        
        stage('BUILD IMAGE'){
            steps{
                echo 'Building the Docker image'
                sh "docker build -t $APP_NAME:$REPO_VERSION $DOCKERFILE_URL"
                sh "docker tag $APP_NAME:$REPO_VERSION $APP_NAME:latest"
                sh 'docker images'
            }
        }
        stage('TEST IMAGE') {
            steps {
                echo 'Testing the image'
                sh 'docker network create test'
                sh 'docker run --rm -dit --name database -e MYSQL_DATABASE=phonebook_db -e MYSQL_USER=admin -e MYSQL_PASSWORD=Talha_123 -e MYSQL_ROOT_PASSWORD=Talha123 --network test  mysql:5.7'
                sh 'sleep 10'
                sh 'docker run --rm --name test -dit -p 80:80 --network test $APP_NAME:$REPO_VERSION'
                sh 'sleep 3'
                sh 'curl -s $(curl -s 169.254.169.254/latest/meta-data/public-ipv4):80 1> /dev/null && echo "Image passed the test"'
                sh 'docker stop test database'
                sh 'docker network rm test'
                echo 'Passed the test'
            }
        }

        stage('PUSH IMAGE'){
            steps {
                echo 'Pushing Docker image to the ECR repo'
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com'
                sh 'docker push $APP_NAME:$REPO_VERSION'
                sh 'docker push $APP_NAME:latest'
            }
        }
        stage('VERIFY IMAGE') {
            steps{
                echo 'Verifying the Image'
                sh 'aws ecr describe-images --repository-name $PROJECT --region us-east-1 | grep $REPO_VERSION && echo "The Image is the ECR repo"'
            }
        }

        stage('CREATE CLUSTER'){
            steps{
                echo 'Downloading the template from GitHub'
                sh 'wget ${GIT_FILE_URL}phonebook_infrastructure_cfn_template.yaml'
                echo 'Creating the cluster for the App'
                sh '''aws cloudformation create-stack --stack-name $PROJECT \
                --template-body file://phonebook_infrastructure_cfn_template.yaml \
                --parameters ParameterKey=KeyPairName,ParameterValue=talha-virginia \
                --region us-east-1 \
                --capabilities CAPABILITY_NAMED_IAM || echo "Using existing cluster"'''
            }
        }

        stage('TEST CLUSTER') {
            steps {
                echo 'Testing the cluster'
                script {
                    while (true) {
                        try {
                            masterDNS = sh(script: '''aws ec2 describe-instances --region us-east-1 --filters "Name=tag:Name,Values='Docker Grand Master of ${PROJECT}'" "Name=instance-state-name,Values=running" --query Reservations[*].Instances[*].[PublicDnsName] --output text''', returnStdout: true).trim()
                            if (masterDNS){
                                echo 'Docker Grand Master is Running'
                                break
                            }
                            else {
                                sh 'sleep 30'
                            }
                        }
                        catch (Exception e) {
                            sh 'echo "Not Ready"'
                        }
                    }
                }
                script {
                    while (true) {
                        instanceID = sh(script: """aws ec2 describe-instances --region us-east-1 --filters "Name=tag:Name,Values='Docker Grand Master of ${PROJECT}'" "Name=instance-state-name,Values=running" --query Reservations[*].Instances[*].[InstanceId] --output text""", returnStdout: true).trim()
                        try {
                            is_dockerUP = sh(script: "curl -s ${masterDNS}:8080", returnStdout: true).trim()
                            if (is_dockerUP){
                                echo 'Docker Service is Running'
                                break
                            }
                        }
                        catch (Exception e) {
                            sh 'echo "Not Ready"'
                            sh 'sleep 30'
                        }
                    }
                }
            }
        }
        stage('DEPLOY APPLICATION') {
            steps {
                echo 'Deploying the application'
                script {
                    sshagent(credentials : ['talha-virginia']) {
                        sh """ssh -t -t -o StrictHostKeyChecking=no ec2-user@${masterDNS} "curl -o 'init.sql' -L ${GIT_FILE_URL}init.sql && \
sleep 3 && \
cat << EOF > docker-compose.yaml -
version: '3.8'
  
services:
  database:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: Talha123
      MYSQL_DATABASE: phonebook_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: Talha_123
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - clarusnet
    configs:
      - source: initialdb
        target: /docker-entrypoint-initdb.d/init.sql
  app-server:
    image: "${APP_NAME}:${REPO_VERSION}"
    deploy:
      mode: global  # Create a container in each node
      update_config:
          parallelism: 2
          delay: 5s
          order: start-first
    ports:
      - "80:80"
    networks:
    - clarusnet


networks:
  clarusnet:
    driver: overlay

volumes:
  db-data:

configs:
  initialdb:
    file: ./init.sql
EOF"
"""
                    }
                    sshagent(credentials : ['talha-virginia']) {
                        sh """ssh -t -t -o StrictHostKeyChecking=no ec2-user@${masterDNS} 'cat docker-compose.yaml | docker stack deploy --with-registry-auth -c - phonebook'"""
                    }
                }
            }
        }
        stage('TEST APPLICATION') {
            steps {
                echo 'Testing the application'
                script {
                    while (true) {
                        try {
                            is_app_running = sh(script: "curl -s ${masterDNS}:80", returnStdout: true).trim()
                            if (is_app_running){
                                echo 'Application is Ready!'
                                break
                            }
                        }
                        catch (Exception e) {
                            sh 'echo "Not Ready"'
                            sh 'sleep 30'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f'
            sh 'docker network prune -f'
            sh 'docker container prune -f'
        }

        failure {
            echo 'Something went wrong. Check the log files'
        }

        success { 
            echo "${PROJECT} is successfully completed!"
            echo "Check your website at http://${masterDNS}:80"
            echo "Check your Visualizor at http://${masterDNS}:8080"
        }
    }
}
