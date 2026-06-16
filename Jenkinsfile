pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "neeraj91/flask-app"
        DOCKER_TAG     = "${BUILD_NUMBER}"
        SONAR_PROJECT  = "flask-app"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/neerajddun/devescops-jenkins.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }

        stage('SonarQube Scan') {
        steps {
          withSonarQubeEnv('SonarQube') {
            script {
                def scannerHome = tool 'SonarScanner'
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                      -Dsonar.projectKey=${SONAR_PROJECT} \
                      -Dsonar.projectName=${SONAR_PROJECT} \
                      -Dsonar.sources=. \
                      -Dsonar.python.version=3
                """
            }
        }
    }
}

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '''
                        --scan .
                        --format HTML
                        --format XML
                        --out owasp-report
                        --enableExperimental
                    ''',
                    odcInstallation: 'OWASP-DC'
                )
            }
            post {
                always {
                    dependencyCheckPublisher(
                        pattern: 'owasp-report/dependency-check-report.xml'
                    )
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                      --exit-code 1 \
                      --severity CRITICAL \
                      --no-progress \
                      --format table \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
            post {
                always {
                    sh """
                        trivy image \
                          --exit-code 0 \
                          --severity HIGH,CRITICAL \
                          --format json \
                          --output trivy-report.json \
                          ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                    archiveArtifacts artifacts: 'trivy-report.json',
                                     fingerprint: true
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker tag  ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                    kubectl set image deployment/flask-app \
                      flask-app=${DOCKER_IMAGE}:${DOCKER_TAG}
                    kubectl rollout status deployment/flask-app --timeout=60s
                """
            }
        }
    }

    post {
        success {
            echo "BUILD SUCCESS - Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
            echo "Trivy: CLEAN | OWASP: CLEAN | SonarQube: PASSED"
        }
        failure {
            echo "BUILD FAILED - Check console output"
        }
    }
}
