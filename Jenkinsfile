pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="026376606405"
        AWS_DEFAULT_REGION="us-east-1" 
        IMAGE_REPO_NAME="myapp"
        IMAGE_TAG="v1.0.0"
        HASH_TAG = sh(returnStdout: true, script: "git rev-parse --short=5 HEAD").trim()
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }
    options {
        ansiColor('xterm')
        timestamps()
        timeout(time: 150, unit: "MINUTES")
    }
    stages {
        //login to ecr
        stage('Logging into AWS ECR') {
            steps {
                withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                    script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    }
                }
                 
            }
        }

        stage('Cloning Git') {
            steps {
                withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/aaron027/my-app.git']]])     
                }
            }
        }
  
    // Building Docker images
        stage('Building image') {
            steps{
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}-${HASH_TAG}"     
                    }
                }
            }
        }

    // Uploading Docker images into AWS ECR
         stage("Pushing to ECR") {
            steps{
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}-${HASH_TAG}"
                        sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}-${HASH_TAG}"
                    }
                }
            }
         }

    // Uploading Docker images into AWS ECR
        stage('Cloning Git from terraform') {
            steps {
                withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/aaron027/tf_junglemeet.git']]])
                }
            }
        }

        stage("start ecs service") {
            steps{
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        dir('backend'){
                            sh '''
                            terraform init
                            terraform validate
                            terraform apply -auto-approve -var=\"image_tag=${IMAGE_TAG}-${HASH_TAG}\"
                            '''
                        }
                    }
                }
            }
        }
        stage("destoy ecs service") {
            steps{
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        dir('backend'){
                            sh '''
                            terraform destroy -auto-approve -var=\"image_tag=${IMAGE_TAG}-${HASH_TAG}\"
                            '''
                        }
                    }
                }
            }
        }
    }
    
}