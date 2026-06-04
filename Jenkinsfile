pipeline {
agent any

```
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
            docker build -t $IMAGE_NAME:$IMAGE_TAG .
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
            docker push $IMAGE_NAME:$IMAGE_TAG
            '''
        }
    }

    stage('Deploy') {
        steps {
            sh '''
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
```

}
