pipeline {
    agent any
    tools {
        maven 'maven-3.6.0'
    }
    environment {
        DOCKER_REPO_SERVER = '718318228454.dkr.ecr.us-east-1.amazonaws.com'
        DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
    }
    stages {
        stage("build jar") {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn package'
                }
            }
        }
        stage("build image") {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'ecr-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                        sh "echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}"
                        sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage("deploy") {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo "deploying the application..."
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/services.yaml | kubectl apply -f -'
                }
            }
        }
    }
}