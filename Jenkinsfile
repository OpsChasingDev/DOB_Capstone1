pipeline {
    agent any
    tools {
        maven 'maven-3.6.0'
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
                    withCredentials([usernamePassword(credentialsId: 'Docker Hub repo', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh 'docker build -t opschasingdev/dockerhub:maven-app-2.0 .'
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh 'docker push opschasingdev/dockerhub:maven-app-2.0'
                    }
                }
            }
        }
        stage("deploy") {
            steps {
                script {
                    echo "deploying the application..."
                }
            }
        }
    }
}