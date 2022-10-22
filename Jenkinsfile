pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="026376606405"
        AWS_DEFAULT_REGION="us-east-1" 
        IMAGE_REPO_NAME="myapp"
        IMAGE_TAG= sh(returnStdout: true, script: "git rev-parse --short=5 HEAD").trim()
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        SERVICE_NAME = 'junglemeet-service-dev'
        TASK_FAMILY="junglemeet-task-dev" // at least one container needs to have the same name as the task definition
        DESIRED_COUNT="2"
        CLUSTER_NAME = "junglemeet-cluster-dev"
        EXECUTION_ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole"
        AWS_ECS_TASK_DEFINITION_PATH = './taskdef_template.json'
    }
    options {
        ansiColor('xterm')
        timestamps()
        timeout(time: 150, unit: "MINUTES")
    }
    def taskDefile      = "file://taskdef_template-${IMAGE_TAG}.json"
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
                        // sh "ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --force-new-deployment --region ${AWS_DEFAULT_REGION} --task-definition"

                        sh  "                                                                     \
          sed -e  's;%IMAGE_TAG%;${IMAGE_TAG};g'                             \
                  taskdef_template.json >                                      \
                  taskdef_template.json-${IMAGE_TAG}.json                      \
        "

        // Get current [TaskDefinition#revision-number]
        def currTaskDef = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs describe-task-definition  --task-definition ${TASK_FAMILY}     \
                                              | egrep 'revision'                  \
                                              | tr ',' ' '                        \
                                              | awk '{print \$2}'                 \
          "
        ).trim()

        def currentTask = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs list-tasks  --cluster ${CLUSTER_NAME}                          \
                                --family ${TASK_FAMILY}                            \
                                --output text                                     \
                                | egrep 'TASKARNS'                                \
                                | awk '{print \$2}'                               \
          "
        ).trim()

        /*
        / Scale down the service
        /   Note: specifiying desired-count of a task-definition in a service -
        /   should be fine for scaling down the service, and starting a new task,
        /   but due to the limited resources (Only one VM instance) is running
        /   there will be a problem where one container is already running/VM,
        /   and using a port(80/443), then when trying to update the service -
        /   with a new task, it will complaine as the port is already being used,
        /   as long as scaling down the service/starting new task run simulatenously
        /   and it is very likely that starting task will run before the scaling down service finish
        /   so.. we need to manually stop the task via aws ecs stop-task.
        */
        if(currTaskDef) {
          sh  "                                                                   \
            aws ecs update-service  --cluster ${CLUSTER_NAME}                      \
                                    --service ${SERVICE_NAME}                      \
                                    --task-definition ${TASK_FAMILY}:${currTaskDef}\
                                    --desired-count 0                             \
          "
        }
        if (currentTask) {
          sh "aws ecs stop-task --cluster ${CLUSTER_NAME} --task ${currentTask}"
        }

        // Register the new [TaskDefinition]
        sh  "                                                                     \
          aws ecs register-task-definition  --family ${TASK_FAMILY}                \
                                            --cli-input-json ${taskDefile}        \
        "

        // Get the last registered [TaskDefinition#revision]
        def taskRevision = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs describe-task-definition  --task-definition ${TASK_FAMILY}     \
                                              | egrep 'revision'                  \
                                              | tr ',' ' '                        \
                                              | awk '{print \$2}'                 \
          "
        ).trim()

        // ECS update service to use the newly registered [TaskDefinition#revision]
        //
        sh  "                                                                     \
          aws ecs update-service  --cluster ${CLUSTER_NAME}                        \
                                  --service ${SERVICE_NAME}                        \
                                  --task-definition ${TASK_FAMILY}:${taskRevision} \
                                  --desired-count 1                               \
        "
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