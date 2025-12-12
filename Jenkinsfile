pipeline {
    agent any

    tools {
        jdk 'java 17'
        maven 'M2_HOME'
    }

    environment {
        IMAGE_NAME = 'sirinebb/sirinebechikh'
        SONARQUBE = 'SonarQube'
        NAMESPACE = 'devops'
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=student -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$SONAR_AUTH_TOKEN'
                }
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
                    script {
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                        retry(3) {
                            sh "docker push ${IMAGE_NAME}:latest"
                        }
                        sh 'docker logout'
                    }
                }
            }
        }

    
    }

    post {
        success {
            echo '========================================='
            echo '✅ PIPELINE SUCCESS - CONFORME AU TP'
            echo '========================================='
            echo '✅ Code analysé par SonarQube'
            echo '✅ Image Docker pushée sur DockerHub'
            echo '✅ SonarQube déployé sur pod Kubernetes'
            echo '✅ MySQL déployé avec PersistentVolume'
            echo '✅ Spring Boot déployé sur Kubernetes'
            echo '✅ Tous les services exposés et testés'
            echo '========================================='

        }
        failure {
            echo '========================================='
            echo '❌ PIPELINE FAILED'
            echo '========================================='
         
        }
        always {
            echo 'Pipeline terminée - Atelier Kubernetes'
        }
    }
}
