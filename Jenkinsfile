pipeline {
    agent any
    
    environment {
        GITHUB_USER = 'areppzubaidi'
        IMAGE_NAME = 'flask-hello-world'
        IMAGE_TAG = "ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
        LATEST_TAG = "ghcr.io/${GITHUB_USER}/${IMAGE_NAME}:latest"
        K8S_NAMESPACE = 'flask-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Test Application') {
            steps {
                script {
                    sh 'python3 -m pip install -r requirements.txt'
                    sh '''
                        python3 -c "
                        from app import app
                        import sys
                        with app.test_client() as client:
                            response = client.get('/')
                            assert response.status_code == 200
                            print('✓ Root endpoint test passed')
                            
                            health_response = client.get('/health')
                            assert health_response.status_code == 200
                            print('✓ Health endpoint test passed')
                            print('All tests passed!')
                        "
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_TAG} ."
                    sh "docker tag ${IMAGE_TAG} ${LATEST_TAG}"
                }
            }
        }
        
        stage('Push to GHCR') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'ghcr-credentials',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )]) {
                        sh """
                            echo \$GITHUB_TOKEN | docker login ghcr.io -u \$GITHUB_USERNAME --password-stdin
                            docker push ${IMAGE_TAG}
                            docker push ${LATEST_TAG}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to K3s') {
            steps {
                script {
                    sh """
                        # Create namespace if not exists
                        kubectl apply -f k8s/namespace.yaml
                        
                        # Deploy application
                        kubectl apply -f k8s/service.yaml
                        
                        # Update deployment with new image
                        kubectl set image deployment/flask-hello-world flask-app=${IMAGE_TAG} -n ${K8S_NAMESPACE} --record || \
                        kubectl apply -f k8s/deployment.yaml
                        
                        # Wait for rollout
                        kubectl rollout status deployment/flask-hello-world -n ${K8S_NAMESPACE} --timeout=300s
                        
                        # Verify deployment
                        echo "=== Deployment Status ==="
                        kubectl get deployments -n ${K8S_NAMESPACE}
                        echo "=== Pods Status ==="
                        kubectl get pods -n ${K8S_NAMESPACE}
                        echo "=== Services ==="
                        kubectl get services -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
        
        stage('Smoke Test') {
            steps {
                script {
                    sh """
                        # Wait for pods to be ready
                        sleep 30
                        
                        # Get a pod name
                        POD_NAME=\$(kubectl get pods -n ${K8S_NAMESPACE} -l app=flask-hello-world -o jsonpath='{.items[0].metadata.name}')
                        
                        # Test the application
                        kubectl exec -n ${K8S_NAMESPACE} \$POD_NAME -- curl -s http://localhost:5000/ || echo "Direct curl failed, continuing..."
                        
                        # Test via port-forward
                        timeout 30s kubectl port-forward -n ${K8S_NAMESPACE} service/flask-hello-world-service 8080:80 &
                        sleep 5
                        curl -s http://localhost:8080/ || echo "Port-forward test failed"
                        pkill -f "kubectl port-forward"
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "=== Cleaning up ==="
            sh "docker rmi ${IMAGE_TAG} ${LATEST_TAG} || true"
        }
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
