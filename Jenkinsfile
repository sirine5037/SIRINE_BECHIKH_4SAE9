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
        stage('Declarative: Checkout SCM') {
            steps {
                echo '=== Checkout du code source ==='
                checkout scm
            }
        }
        
        stage('Declarative: Tool Install') {
            steps {
                echo '=== Vérification des outils ==='
                sh '''
                    echo "Java version:"
                    java -version
                    echo ""
                    echo "Maven version:"
                    mvn -version
                '''
            }
        }
        
        stage('Checkout') {
            steps {
                echo '=== Code source récupéré ==='
                sh 'ls -la'
            }
        }
        
        stage('Run Tests') {
            steps {
                echo '=== Exécution des tests ==='
                script {
                    try {
                        sh 'mvn test'
                    } catch (Exception e) {
                        echo "⚠️ Warning: Tests failed - ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                echo '=== Build Maven ==='
                script {
                    try {
                        // Nettoyage et build
                        sh 'mvn clean package -DskipTests'
                        
                        // Vérifier que le JAR existe
                        sh '''
                            echo "Contenu du dossier target:"
                            ls -lh target/
                            echo ""
                            echo "Fichiers JAR trouvés:"
                            find target/ -name "*.jar" -type f
                        '''
                        
                        // Vérification explicite
                        def jarExists = sh(
                            script: 'test -f target/student-management-0.0.1-SNAPSHOT.jar && echo "true" || echo "false"',
                            returnStdout: true
                        ).trim()
                        
                        if (jarExists == "false") {
                            error "❌ JAR file not found!"
                        }
                        
                        echo "✅ JAR créé avec succès"
                        
                    } catch (Exception e) {
                        echo "❌ Build failed: ${e.message}"
                        sh 'cat target/surefire-reports/*.txt || true'
                        error "Build Maven a échoué"
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo '=== Analyse SonarQube ==='
                script {
                    try {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                mvn sonar:sonar \
                                -Dsonar.projectKey=student \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.login=${SONAR_AUTH_TOKEN}
                            '''
                        }
                        echo "✅ Analyse SonarQube terminée"
                    } catch (Exception e) {
                        echo "⚠️ SonarQube analysis failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Archive Artifact') {
            steps {
                echo '=== Archivage des artifacts ==='
                script {
                    try {
                        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                        echo "✅ Artifact archivé"
                    } catch (Exception e) {
                        echo "⚠️ Archive failed: ${e.message}"
                    }
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                echo '=== Build de l\'image Docker ==='
                script {
                    try {
                        // Vérifier Dockerfile
                        sh '''
                            echo "Vérification du Dockerfile:"
                            if [ ! -f Dockerfile ]; then
                                echo "❌ Dockerfile not found!"
                                exit 1
                            fi
                            cat Dockerfile
                            echo ""
                        '''
                        
                        // Vérifier que le JAR existe avant le build Docker
                        sh '''
                            if [ ! -f target/student-management-0.0.1-SNAPSHOT.jar ]; then
                                echo "❌ JAR not found for Docker build!"
                                exit 1
                            fi
                        '''
                        
                        // Build Docker
                        sh """
                            docker build -t ${IMAGE_NAME}:latest .
                            docker tag ${IMAGE_NAME}:latest ${IMAGE_NAME}:${BUILD_NUMBER}
                        """
                        
                        echo "✅ Image Docker créée"
                        
                        // Lister les images
                        sh "docker images | grep ${IMAGE_NAME} || true"
                        
                    } catch (Exception e) {
                        echo "❌ Docker build failed: ${e.message}"
                        error "Build Docker a échoué"
                    }
                }
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                echo '=== Push vers DockerHub ==='
                script {
                    try {
                        withCredentials([usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh '''
                                echo "Connexion à DockerHub..."
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                
                                echo "Push de l'image latest..."
                                docker push ${IMAGE_NAME}:latest
                                
                                echo "Push de l'image ${BUILD_NUMBER}..."
                                docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                                
                                echo "Déconnexion..."
                                docker logout
                            '''
                        }
                        echo "✅ Images pushées sur DockerHub"
                    } catch (Exception e) {
                        echo "❌ Docker push failed: ${e.message}"
                        error "Push Docker a échoué"
                    }
                }
            }
        }
        
        stage('Declarative: Post Actions') {
            steps {
                echo '=== Nettoyage ==='
                sh """
                    docker rmi ${IMAGE_NAME}:latest || true
                    docker rmi ${IMAGE_NAME}:${BUILD_NUMBER} || true
                """
            }
        }
    }
    
    post {
        success {
            echo '========================================='
            echo '✅ PIPELINE SUCCESS'
            echo '========================================='
            echo "✅ Code analysé par SonarQube"
            echo "✅ Image Docker: ${IMAGE_NAME}:${BUILD_NUMBER}"
            echo "✅ Image pushée sur DockerHub"
            echo '========================================='
        }
        
        failure {
            echo '========================================='
            echo '❌ PIPELINE FAILED'
            echo '========================================='
            echo 'Vérifiez les logs ci-dessus pour plus de détails'
            echo '========================================='
        }
        
        unstable {
            echo '========================================='
            echo '⚠️ PIPELINE UNSTABLE'
            echo '========================================='
            echo 'Certaines étapes ont échoué mais le build continue'
            echo '========================================='
        }
        
        always {
            echo 'Nettoyage du workspace...'
            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true,
                patterns: [
                    [pattern: 'target/**', type: 'INCLUDE']
                ]
            )
        }
    }
}
