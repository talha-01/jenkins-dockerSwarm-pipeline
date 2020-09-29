pipeline {
    agent {
        label 'master'
    }

    environment {
        PROJECT="project-204"
        APP_NAME="282594292356.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}"
        REPO_VERSION="App-${BUILD_NUMBER}"
        DOCKERFILE_URL="https://github.com/talha-01/project-203-docker-swarm.git"
    }


    stages {
        stage('testing') {
            steps {
                echo 'test'
                sh 'echo test'
            }
        }


        stage('creating ECR Repo'){
            steps{
                echo 'creating ECR Repo'
                sh '''
                aws ecr create-repository \
                  --repository-name ${PROJECT} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region us-east-1
                '''                
            }
        }
        stage('checking the ECR Repo') {
            steps{
                echo 'Checking the ECR Repo'
                sh 'aws ecr describe-repositories | grep $PROJECT && echo "ECR Repo created"'
            }
        }
        
        stage('building Docker image'){
            steps{
                echo 'Building Docker image'
                sh "docker build -t $APP_NAME:$REPO_VERSION $DOCKERFILE_URL"
                sh "docker tag $APP_NAME:$REPO_VERSION $APP_NAME:latest"
            }
        }
        stage('Listing images') {
            steps{
                echo 'Listing Images'
                sh 'docker images'
            }
        }

        stage('pushing Docker image to ECR Repo'){
            steps {
                echo 'Pushing Docker image to ECR Repo'
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 282594292356.dkr.ecr.us-east-1.amazonaws.com'
                sh 'docker push $APP_NAME:$REPO_VERSION'
                sh 'docker push $APP_NAME:latest'
            }
        }
        stage('checking the images') {
            steps{
                echo 'Checking the Image'
                sh 'aws ecr describe-images --repository-name $PROJECT | grep $REPO_VERSION && echo "The Image verified"'
            }
        }

        stage('creating infrastructure for the app'){
            steps{
                echo 'Downloading the template from GitHub'
                sh 'wget https://raw.githubusercontent.com/talha-01/project-203-docker-swarm/master/phonebook_infrastructure_cfn_template.yaml'
                echo 'Creating Infrastructure for the App'
                sh '''aws cloudformation create-stack --stack-name my_stack \
                --template-body file://phonebook_infrastructure_cfn_template.yaml \
                --parameters ParameterKey=KeyPairName,ParameterValue=talha-virginia 
                --capabilities CAPABILITY_NAMED_IAM'''
                // script {
                //     int counter = 0 ;

                //     while ( counter < 3 ) {
                //         println('Counting... ' +counter);
                //         sleep(2)
                //         counter++;

                //         //sh 'aws ec2 describe ...' get ip
                //         //if (ip.length 7 ) ... break
                //         //else sleep(10)
                //     }
                // }
            }
        }

        // stage('test the Viz App') {
        //     steps {
        //         echo 'test the Viz App'
        //         script {
        //             int counter = 0 ;

        //             while ( counter < 3 ) {
        //                 println('Counting... ' +counter);
        //                 sleep(2)
        //                 counter++;
        //                 //try catch block with groovy/java
        //                 //sh 'curl ip:8080 ...' break
        //                 //failure ... sleep(5)
        //             }
        //         }
        //     }
        // }

        // stage('deploying the application') {
        //     steps {
        //         echo 'deploying the application'
        //         sh 'mssh .... git clone'
        //         sh 'mssh .... docker stack deploy'
        //     }
        // }

        // stage('test phonebook the application') {
        //     steps {
        //         echo 'test phonebook the application'
        //         script {
        //             int counter = 0 ;

        //             while ( counter < 3 ) {
        //                 println('Counting... ' +counter);
        //                 sleep(2)
        //                 counter++;
        //                 //try catch block with groovy/java
        //                 //sh 'curl ip:8080 ...' break
        //                 //failure ... sleep(5)
        //             }
        //         }
        //     }
        // }
    }

    post {
        always {
            echo 'Goodbye ALL... Please come back soon'
        }

        failure {
            echo 'Sorry but you messed up...'
        }

        success { 
            echo 'You are the man/woman...'
        }
    }
}
