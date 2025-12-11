pipeline {
    agent any

    tools {
        jdk 'java 17'
        maven 'M2_HOME'
    }

    environment {
        IMAGE_NAME = 'sirinebb/sirinebechikh'
    }

    stages {



        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Run Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }



        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:latest .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                       echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                       docker push ${IMAGE_NAME}:latest
                       docker logout
                   '''
               }
           }
       }
    }



    post {
        success {
            echo 'Pipeline SUCCESS ✔'
        }
        failure {
            echo 'Pipeline FAILED ❌'
        }
    }
}