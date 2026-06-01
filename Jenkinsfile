pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t neeraj91/flask-app:latest .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push neeraj91/flask-app:latest
                    '''
                }
            }
        }

        stage('Run Flask App') {
            steps {
                sh '''
                    docker stop flask-app || true
                    docker rm flask-app || true
                    docker run -d \
                        --name flask-app \
                        -p 5000:5000 \
                        neeraj91/flask-app:latest
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    sleep 2
                    echo "Hit http://localhost:5000 to see the app"
                '''
            }
        }

        stage("Cleanup") {
            steps {
                cleanWs()
            }
        }
    }
}