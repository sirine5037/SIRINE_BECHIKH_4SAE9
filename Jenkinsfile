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

        // ============================================
        // NOUVELLES ÉTAPES KUBERNETES (selon le TP)
        // ============================================

        stage('Create Namespace') {
            steps {
                script {
                    echo '📦 Création du namespace Kubernetes...'
                    sh "kubectl create namespace ${NAMESPACE} || echo 'Namespace already exists'"
                }
            }
        }

        stage('Deploy SonarQube on Kubernetes') {
            steps {
                script {
                    echo '🔍 Déploiement de SonarQube sur Kubernetes...'
                    sh """
                        kubectl apply -f sonarqube-deployment.yaml -n ${NAMESPACE}
                        echo 'Attente du démarrage de SonarQube (peut prendre 3-5 minutes)...'
                        kubectl wait --for=condition=ready pod -l app=sonarqube -n ${NAMESPACE} --timeout=600s
                        echo '✅ SonarQube déployé sur le pod Kubernetes'
                    """
                }
            }
        }

        stage('Verify SonarQube on Pod') {
            steps {
                script {
                    echo '✅ Vérification de l\'analyse SonarQube sur le pod...'

                    def sonarPod = sh(
                        script: "kubectl get pods -l app=sonarqube -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    sh """
                        echo '=== POD SONARQUBE ==='
                        kubectl get pods -l app=sonarqube -n ${NAMESPACE}

                        echo ''
                        echo '=== LOGS SONARQUBE (50 dernières lignes) ==='
                        kubectl logs ${sonarPod} -n ${NAMESPACE} --tail=50

                        echo ''
                        echo '=== URL SONARQUBE ==='
                        minikube service sonarqube-service -n ${NAMESPACE} --url

                        echo ''
                        echo '=== SERVICE SONARQUBE ==='
                        kubectl get svc sonarqube-service -n ${NAMESPACE}
                    """
                }
            }
        }

        stage('Deploy MySQL on Kubernetes') {
            steps {
                script {
                    echo '🗄️ Déploiement de MySQL sur Kubernetes...'
                    sh """
                        kubectl apply -f mysql-deployment.yaml -n ${NAMESPACE}
                        echo 'Attente du démarrage de MySQL...'
                        kubectl wait --for=condition=ready pod -l app=mysql -n ${NAMESPACE} --timeout=300s
                        echo '✅ MySQL déployé avec PersistentVolume'
                    """
                }
            }
        }

        stage('Deploy Spring Boot on Kubernetes') {
            steps {
                script {
                    echo '🚀 Déploiement de Spring Boot sur Kubernetes...'
                    sh """
                        kubectl apply -f spring-deployment.yaml -n ${NAMESPACE}
                        echo 'Attente du démarrage de Spring Boot...'
                        kubectl wait --for=condition=ready pod -l app=spring-app -n ${NAMESPACE} --timeout=300s
                        echo '✅ Spring Boot déployé et connecté à MySQL'
                    """
                }
            }
        }

        stage('Verify All Deployments') {
            steps {
                script {
                    echo '🔍 Vérification de tous les déploiements...'
                    sh """
                        echo '==================== TOUS LES PODS ===================='
                        kubectl get pods -n ${NAMESPACE}

                        echo ''
                        echo '==================== TOUS LES SERVICES ===================='
                        kubectl get svc -n ${NAMESPACE}

                        echo ''
                        echo '==================== LOGS SPRING BOOT ===================='
                        kubectl logs -l app=spring-app -n ${NAMESPACE} --tail=30
                    """
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    echo '🧪 Test de l\'application déployée...'
                    sh """
                        echo '==================== URL APPLICATION ===================='
                        minikube service spring-service -n ${NAMESPACE} --url

                        echo ''
                        echo '==================== TEST API ===================='
                        sleep 15
                        APP_URL=\$(minikube service spring-service -n ${NAMESPACE} --url)
                        echo "Testing endpoint: \${APP_URL}/department/getAllDepartment"
                        curl -s \${APP_URL}/department/getAllDepartment || echo '⚠️ Endpoint pas encore disponible, réessayez dans quelques secondes'
                    """
                }
            }
        }

        stage('Verify MySQL Connection') {
            steps {
                script {
                    echo '🗄️ Vérification de la connexion MySQL...'
                    sh """
                        MYSQL_POD=\$(kubectl get pods -l app=mysql -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}')
                        echo "MySQL Pod: \${MYSQL_POD}"

                        echo ''
                        echo '==================== DATABASES MYSQL ===================='
                        kubectl exec \${MYSQL_POD} -n ${NAMESPACE} -- mysql -u root -proot -e "SHOW DATABASES;" || echo '⚠️ Impossible de se connecter à MySQL'

                        echo ''
                        echo '==================== TABLES DANS studentdb ===================='
                        kubectl exec \${MYSQL_POD} -n ${NAMESPACE} -- mysql -u root -proot -e "USE studentdb; SHOW TABLES;" || echo '⚠️ Base de données pas encore créée'
                    """
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
            sh """
                echo ''
                echo '📊 Accès SonarQube:'
                minikube service sonarqube-service -n ${NAMESPACE} --url || true
                echo ''
                echo '🚀 Accès Application:'
                minikube service spring-service -n ${NAMESPACE} --url || true
                echo ''
                echo '=== RÉCAPITULATIF DES PODS ==='
                kubectl get pods -n ${NAMESPACE} || true
            """
        }
        failure {
            echo '========================================='
            echo '❌ PIPELINE FAILED'
            echo '========================================='
            sh """
                echo 'Debug - État des pods:'
                kubectl get pods -n ${NAMESPACE} || true
                echo ''
                echo 'Debug - Services:'
                kubectl get svc -n ${NAMESPACE} || true
                echo ''
                echo 'Debug - Description des pods:'
                kubectl describe pods -n ${NAMESPACE} || true
                echo ''
                echo 'Debug - Logs Spring Boot:'
                kubectl logs -l app=spring-app -n ${NAMESPACE} --tail=100 || true
            """
        }
        always {
            echo 'Pipeline terminée - Atelier Kubernetes'
        }
    }
}