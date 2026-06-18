pipeline {
    agent { label 'worker-node' }

    environment {
        DOCKER_IMAGE  = "neeraj91/flask-app"
        DOCKER_TAG    = "${BUILD_NUMBER}"
        SONAR_PROJECT = "flask-app"
    }

    stages {

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
                sleep(time: 15, unit: 'SECONDS')

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push $DOCKER_IMAGE:$DOCKER_TAG

                        docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest

                        docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    kubectl set image deployment/flask-app \
                        flask-app=${DOCKER_IMAGE}:${DOCKER_TAG}

                    for i in 1 2 3; do
                        kubectl rollout status deployment/flask-app --timeout=120s && break
                        echo "Retrying rollout status..."
                        sleep 10
                    done

                    kubectl wait \
                        --for=condition=available \
                        deployment/flask-app \
                        --timeout=300s

                    echo "Running Image:"
                    kubectl get deployment flask-app \
                        -o=jsonpath='{.spec.template.spec.containers[0].image}'

                    echo
                """
            }
        }
    }

    post {

        success {
            echo "BUILD SUCCESS - Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
            echo "SonarQube: PASSED"
        }

        unstable {
            echo "BUILD UNSTABLE - Review OWASP and Trivy reports"
        }

        failure {
            echo "BUILD FAILED - Check console logs"
        }

        always {
            cleanWs()
        }
    }
}