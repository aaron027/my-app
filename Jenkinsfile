pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="026376606405"
        AWS_DEFAULT_REGION="us-east-1" 
        IMAGE_REPO_NAME="myapp"
        IMAGE_TAG= sh(returnStdout: true, script: "git rev-parse --short=5 HEAD").trim()
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        SERVICE_NAME = 'junglemeet-service-dev'
        TASK_FAMILY="junglemeet-task-dev" 
        TASKDEF_NAME = "junglemeet-container-dev"
        DESIRED_COUNT="2"
        CLUSTER_NAME = "junglemeet-cluster-dev"
        EXECUTION_ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole"
        AWS_ECS_TASK_DEFINITION_PATH = 'file://taskdef_template.json'
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
                        dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"     
                    }
                }
            }
        }

    // Uploading Docker images into AWS ECR
         stage("Pushing to ECR") {
            steps{
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
                        sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                    }
                }
            }
         }

        stage('Deploy Image to ECS') {
            steps{
                // update service
                script {
                    withAWS(credentials: 'AWS_Credentials', region: 'us-east-1') {
                        def newimageurl = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                        def oldimageurl = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
                        sh "sed -i -e 's#${oldimageurl}#${newimageurl}#' ./taskdef_template.json"
                        sh "aws ecs register-task-definition --family ${TASK_FAMILY} --cli-input-json file://${WORKSPACE}/taskdef_template.json --region ${AWS_DEFAULT_REGION}"
                        sh "aws ecs describe-task-definition --task-definition ${TASKDEF_NAME} --region ${AWS_DEFAULT_REGION} | jq .taskDefinition.revision > output.txt"
                        def REVISION = readFile "${WORKSPACE}/output.txt"
                        // updated_task_definition_revision=$(echo "$updated_task_definition_info" | jq '.taskDefinition.revision')
                        sh "aws ecs update-service --cluster ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION} --service ${SERVICE_NAME} --task-definition ${TASKDEF_NAME}:$REVISION"
                    }
                }
            }
        }
      
    }
    post {
        always{
            echo 'I will always say Hello again! test for webhook'
            cleanWs()
        }
    }
    
}