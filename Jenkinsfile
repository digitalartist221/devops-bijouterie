pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'digitalartistsn'
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/bijouterie-frontend"
        BACKEND_IMAGE = "${DOCKERHUB_USER}/bijouterie-backend"
        MYSQL_IMAGE = "${DOCKERHUB_USER}/bijouterie-mysql"
    }

    stages {
        // Jenkins recupere le depot Git configure dans le job Pipeline.
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Construction des trois images avec un nom temporaire local.
        stage('Build') {
            steps {
                sh 'docker build -t bijouterie-frontend:build frontend/'
                sh 'docker build -t bijouterie-backend:build backend/'
                sh 'docker build -t bijouterie-mysql:build database/'
            }
        }

        // Chaque image recoit le numero du build et un second tag latest.
        stage('Tag') {
            steps {
                sh 'docker tag bijouterie-frontend:build $FRONTEND_IMAGE:$BUILD_NUMBER'
                sh 'docker tag bijouterie-backend:build $BACKEND_IMAGE:$BUILD_NUMBER'
                sh 'docker tag bijouterie-mysql:build $MYSQL_IMAGE:$BUILD_NUMBER'
                sh 'docker tag bijouterie-frontend:build $FRONTEND_IMAGE:latest'
                sh 'docker tag bijouterie-backend:build $BACKEND_IMAGE:latest'
                sh 'docker tag bijouterie-mysql:build $MYSQL_IMAGE:latest'
            }
        }

        // Le token reste dans le credential Jenkins et sa valeur est masquee.
        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_LOGIN',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh 'echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_LOGIN" --password-stdin'
                    sh 'docker push $FRONTEND_IMAGE:$BUILD_NUMBER'
                    sh 'docker push $BACKEND_IMAGE:$BUILD_NUMBER'
                    sh 'docker push $MYSQL_IMAGE:$BUILD_NUMBER'
                    sh 'docker push $FRONTEND_IMAGE:latest'
                    sh 'docker push $BACKEND_IMAGE:latest'
                    sh 'docker push $MYSQL_IMAGE:latest'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}
