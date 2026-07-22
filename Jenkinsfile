pipeline {
    agent none

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        DOCKERHUB_USERNAME    = "purushothdoc"
        BACKEND_IMAGE         = "${DOCKERHUB_USERNAME}/notes-backend:${env.BUILD_NUMBER}"
        FRONTEND_IMAGE        = "${DOCKERHUB_USERNAME}/notes-frontend:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                checkout scm
                stash name: 'source', includes: '**'
            }
        }

        stage('Build & Push Images') {
            agent { label 'built-in' }
            steps {
                unstash 'source'
                sh '''
                    echo "$DOCKERHUB_CREDENTIALS_PSW" | docker login -u "$DOCKERHUB_CREDENTIALS_USR" --password-stdin

                    docker build -t $BACKEND_IMAGE -t $DOCKERHUB_USERNAME/notes-backend:latest ./backend
                    docker push $BACKEND_IMAGE
                    docker push $DOCKERHUB_USERNAME/notes-backend:latest

                    docker build -t $FRONTEND_IMAGE -t $DOCKERHUB_USERNAME/notes-frontend:latest ./frontend
                    docker push $FRONTEND_IMAGE
                    docker push $DOCKERHUB_USERNAME/notes-frontend:latest

                    docker logout
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            agent { label 'k8s-master' }
            steps {
                unstash 'source'
                sh '''
                    sed -i "s#<DOCKERHUB_USERNAME>/notes-backend:latest#$BACKEND_IMAGE#g" k8s/backend-deployment.yaml
                    sed -i "s#<DOCKERHUB_USERNAME>/notes-frontend:latest#$FRONTEND_IMAGE#g" k8s/frontend-deployment.yaml

                    kubectl apply -f k8s/postgres-secret.yaml
                    kubectl apply -f k8s/postgres-statefulset.yaml
                    kubectl apply -f k8s/postgres-service.yaml
                    kubectl apply -f k8s/backend-deployment.yaml
                    kubectl apply -f k8s/backend-service.yaml
                    kubectl apply -f k8s/frontend-deployment.yaml
                    kubectl apply -f k8s/frontend-service.yaml

                    kubectl rollout status deployment/backend-deployment --timeout=120s
                    kubectl rollout status deployment/frontend-deployment --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo "Deployed successfully. Frontend accessible on NodePort 30010."
        }
        failure {
            echo "Pipeline failed. Check stage logs above."
        }
    }
}
