pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "samarthfunde"
        FRONTEND_IMAGE = "weather-frontend-image"
        BACKEND_IMAGE  = "weather-backend-image"
        NAMESPACE      = "weather-app"
        SONARQUBE_ENV  = "sonar-server"   // Name configured in Jenkins
    }

    tools {
        nodejs "nodejs"   // Jenkins Global Tool name
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // ---------------- SONARQUBE ----------------

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh """
                    sonar-scanner \
                    -Dsonar.projectKey=weather-app \
                    -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ---------------- DOCKER BUILD ----------------

        stage('Build Docker Images') {
            steps {
                sh """
                docker build -t $DOCKERHUB_REPO/$FRONTEND_IMAGE:${BUILD_NUMBER} ./frontend
                docker build -t $DOCKERHUB_REPO/$BACKEND_IMAGE:${BUILD_NUMBER} ./backend
                """
            }
        }

        // ---------------- TRIVY SCAN ----------------

        stage('Trivy Scan Images') {
            steps {
                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL \
                $DOCKERHUB_REPO/$FRONTEND_IMAGE:${BUILD_NUMBER}

                trivy image --exit-code 1 --severity HIGH,CRITICAL \
                $DOCKERHUB_REPO/$BACKEND_IMAGE:${BUILD_NUMBER}
                """
            }
        }

        // ---------------- PUSH ----------------

        stage('Push Images to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker push $DOCKERHUB_REPO/$FRONTEND_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKERHUB_REPO/$BACKEND_IMAGE:${BUILD_NUMBER}
                    """
                }
            }
        }

        // ---------------- DEPLOY ----------------

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                kubectl set image deployment/frontend \
                frontend-container=$DOCKERHUB_REPO/$FRONTEND_IMAGE:${BUILD_NUMBER} \
                -n $NAMESPACE

                kubectl set image deployment/backend \
                backend-container=$DOCKERHUB_REPO/$BACKEND_IMAGE:${BUILD_NUMBER} \
                -n $NAMESPACE

                kubectl rollout status deployment/frontend -n $NAMESPACE
                kubectl rollout status deployment/backend -n $NAMESPACE
                """
            }
        }
    }

    post {
        always {
            sh "docker system prune -f"
        }
    }
}
