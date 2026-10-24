pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_VERSION', choices: ['blue', 'green'], description: 'Version to route traffic to')
    }

    environment {
        // Change this to your Docker Hub repository if different.
        IMAGE_NAME = 'joash-m/vle8-blue-green'
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        KUBECONFIG_CREDENTIALS = 'kubeconfig'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER -t $IMAGE_NAME:latest .'
            }
        }

        stage('Login and Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASSWORD'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push "$IMAGE_NAME:$BUILD_NUMBER"
                        docker push "$IMAGE_NAME:latest"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Blue and Green') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        kubectl apply -f deployment-blue.yaml
                        kubectl apply -f deployment-green.yaml
                        kubectl set image deployment/blue-deployment web="$IMAGE_NAME:$BUILD_NUMBER"
                        kubectl set image deployment/green-deployment web="$IMAGE_NAME:$BUILD_NUMBER"
                        kubectl rollout status deployment/blue-deployment --timeout=120s
                        kubectl rollout status deployment/green-deployment --timeout=120s
                    '''
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        kubectl apply -f service.yaml
                        kubectl patch service vle8-service --type merge -p '{"spec":{"selector":{"app":"vle8-demo","version":"'"$DEPLOY_VERSION"'"}}}'
                        kubectl get deployments,pods,service vle8-service
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Blue-green deployment completed. Traffic is routed to: ${params.DEPLOY_VERSION}"
        }
        failure {
            echo 'Pipeline failed. Review the stage logs and Jenkins credentials.'
        }
    }
}
