pipeline {
    agent any

    environment {
        IMAGE_NAME = "likith0129/registry-tracker"
        IMAGE_TAG = "latest"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Workspace') {
            steps {
                sh '''
                pwd
                ls -la
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
        stage('AWS ECR Login') {
    steps {
        withCredentials([
            string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
            sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

            aws ecr get-login-password --region ap-south-1 | \
            docker login --username AWS --password-stdin \
            790505843920.dkr.ecr.ap-south-1.amazonaws.com
            '''
        }
    }
}

stage('Tag Image For ECR') {
    steps {
        sh '''
        docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
        790505843920.dkr.ecr.ap-south-1.amazonaws.com/container-registry-tracker:${IMAGE_TAG}
        '''
    }
}

stage('Push To ECR') {
    steps {
        sh '''
        docker push \
        790505843920.dkr.ecr.ap-south-1.amazonaws.com/container-registry-tracker:${IMAGE_TAG}
        '''
    }
}

        stage('Deploy') {
            steps {
                sh '''
                docker rm -f registry-tracker-app || true
                docker rm -f registry-postgres || true
                docker compose down || true
                docker compose pull
                docker compose up -d
                '''
            }
        }
    }

    post {
        success {
            echo 'Docker image pushed and deployed successfully'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
