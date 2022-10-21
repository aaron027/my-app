pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="026376606405"
        AWS_DEFAULT_REGION="us-east-1" 
        IMAGE_REPO_NAME="myapp"
        IMAGE_TAG="latest"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }
    options {
        ansiColor('xterm')
    }
    stages {
        
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
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/sd031/aws_codebuild_codedeploy_nodeJs_demo.git']]])     
                }
            }
        }
  
    // Building Docker images
    stage('Building image') {
      steps{
        script {
          withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
            dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
          }
        }
      }
    }
   
    // Uploading Docker images into AWS ECR
    stage('Pushing to ECR') {
     steps{  
        withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
            script {
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:$IMAGE_TAG"
                    sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
            }
        }
        }
      }
    }
}