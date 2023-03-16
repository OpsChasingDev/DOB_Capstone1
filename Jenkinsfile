// designed to run in EKS cluster - see README for pre-deployment steps both for deploying the EKS cluster and preparing the Jenkins server
pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = '718318228454.dkr.ecr.us-east-1.amazonaws.com'
        DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
    }
    stages {
        stage("increment version") {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage("build jar") {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
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
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo "deploying the application..."
                    // create secret so K8s cluster can have access to private repo
                    withCredentials([usernamePassword(credentialsId: 'ecr-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh 'kubectl create secret docker-registry aws-registry-key \
                            --docker-server=${DOCKER_REPO_SERVER} \
                            --docker-username=$USER \
                            --docker-password=$PASS'
                    }
                    // envsubst is responsible for passing the env vars like $APP_NAME to the yaml files
                    // results are then piped to kubectl apply and passed as a parameter at the end
                    // envsubst must be installed on the Jenkins server and is not out-of-the-box
                    // make available on Jenkins by installing "gettext-base" tool
                    // "apt-get install gettext-base"
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/services.yaml | kubectl apply -f -'
                }
            }
        }
        stage("commit version update") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId:'GitHub_PAT', passwordVariable:'GITHUB_PAT_PASS', usernameVariable:'GITHUB_PAT_USER')]) {
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins-3.droplet"'
                        sh "git remote set-url origin https://${GITHUB_PAT_USER}:${GITHUB_PAT_PASS}@github.com/OpsChasingDev/DOB_Capstone1.git"
                        sh 'git add .'
                        sh 'git commit -m "incrementing app version"'
                        sh 'git push origin main'
                    }
                }
            }
        }
    }
}