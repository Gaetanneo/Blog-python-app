pipeline {
    agent any
    tools {
        git 'Default'
    }
    environment {
        DOCKER_IMAGE_FLASK = "gaetanneo/flask-app"
        DOCKER_IMAGE_MYSQL = "gaetanneo/mysql"
        DOCKER_REGISTRY_CREDENTIALS = "dockerhub-creds"
        NAMESPACE = "jenkins"
        DOCKER_TAG = "${BUILD_NUMBER}"
        PROJECT_NAME = "flask-mysql"
        DOCKER_BUILDKIT = '1'
        REMOTE_HOST = "ubuntu@3.91.184.171"
        REMOTE_K8S_PATH = "/home/ubuntu/k8s"
        DEPLOY_TIMEOUT = "300"
        VERIFY_TIMEOUT = "5"
        KUBE_CONFIG_PATH = "~/.kube/config"
        
    }

    stages {
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/gaetanneo/Blog-python-app.git'
            }
        }

        stage('Clean Environment') {
            steps {
                script {
                    // Clean up any existing containers and volumes
                    sh '''
                        docker-compose down --volumes --remove-orphans
                        docker system prune -f
                    '''
                }
            }
        }

        stage('Run Docker Compose Build') {
            steps {
                script {
                    sh 'docker-compose -f docker-compose.yaml build --no-cache'
                }
            }
        }

        stage('Run Docker Compose Up') {
            steps {
                script {
                    // Start the services
                    sh 'docker-compose up -d'
                    
                    // Verify containers are running
                    sh 'docker-compose ps'
                    
                    // Wait for services to be healthy
                    sh '''
                        # Wait for MySQL to be ready
                        echo "Waiting for MySQL to be ready..."
                        for i in $(seq 1 10); do
                            if docker-compose exec -T mysqldb mysqladmin ping -h localhost --silent; then
                                echo "MySQL is ready!"
                                break
                            fi
                            echo "Waiting for MySQL... attempt $i"
                            sleep 5
                        done
                    '''
                }
            }
        }

        stage('Verify Services') {
            steps {
                script {
                    // Check container status and logs
                    sh '''
                        echo "Checking container status..."
                        docker-compose ps
                        
                        echo "MySQL Container Logs:"
                        docker-compose logs mysql
                        
                        echo "Flask Container Logs:"
                        docker-compose logs flask
                        
                    '''
                }
            }
        }

        stage('Build Flask Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE_FLASK}:${DOCKER_TAG}", ".")
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh '''
                    echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                    docker push ${DOCKER_IMAGE_FLASK}:${DOCKER_TAG}
                    docker tag mysql:latest ${DOCKER_IMAGE_MYSQL}:${DOCKER_TAG}
                    docker push ${DOCKER_IMAGE_MYSQL}:${DOCKER_TAG}
                '''
                }
            }
        }

        
stage('Deploy to Kubernetes') {
    steps {
        sshagent(['k8s']) {
            script {
                try {
                    // Create remote directory and copy files
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 'mkdir -p /home/ubuntu/k8s'
                        scp -o StrictHostKeyChecking=no k8s/* ubuntu@3.91.184.171:/home/ubuntu/k8s/
                    """

                    // Verify and setup kubectl configuration
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                            # Check if kubeconfig exists
                            if [ ! -f ~/.kube/config ]; then
                                mkdir -p ~/.kube
                                sudo cp /etc/kubernetes/admin.conf ~/.kube/config
                                sudo chown -R ubuntu:ubuntu ~/.kube
                            fi
                            
                            # Verify kubectl works
                            kubectl cluster-info

                            # Create namespace if it doesn't exist
                            if ! kubectl get namespace ${NAMESPACE}; then
                                echo "Creating namespace: ${NAMESPACE}"
                                kubectl create namespace ${NAMESPACE}
                            else
                                echo "Namespace ${NAMESPACE} already exists"
                            fi
                        '
                    """

                    // Verify and setup kubectl configuration
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                            # Check if kubeconfig exists
                            if [ ! -f ~/.kube/config ]; then
                                mkdir -p ~/.kube
                                # You might need to adjust this path based on where your actual kubeconfig is stored
                                sudo cp /etc/kubernetes/admin.conf ~/.kube/config
                                sudo chown -R ubuntu:ubuntu ~/.kube
                            fi
                            
                            # Verify kubectl works
                            kubectl cluster-info
                        '
                    """

                    // Define the k8s resources in an array for cleaner iteration
                    def k8sResources = [
                        'deployment-flask.yml',
                        'deployment-mysql.yml',
                        'env-config.yml',
                        'flask-service.yml',
                        'mysql-service.yml',
                        'persistent-volume.yml',
                        'pvc-claim.yml',
                        'sql-inject-config.yml',
                        'storage.yml'
                    ]
                    
                    // Apply all resources with error checking
                    k8sResources.each { resource ->
                        echo "Applying ${resource}..."
                        def result = sh(
                            script: """
                                ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 \
                                'cd /home/ubuntu/k8s && kubectl apply -f ${resource} -n ${NAMESPACE}'
                            """,
                            returnStatus: true
                        )
                        if (result != 0) {
                            error "Failed to apply ${resource}"
                        }
                    }

                    // Parallel deployment status check
                    parallel(
                        'Flask App': {
                            sh """
                                ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 \
                                'kubectl rollout status deployment/flask-app -n ${NAMESPACE} --timeout=300s'
                            """
                        },
                        'MySQL': {
                            sh """
                                ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 \
                                'kubectl rollout status deployment/mysql -n ${NAMESPACE} --timeout=300s'
                            """
                        }
                    )

                } catch (Exception e) {
                    // Log the error details
                    echo "Deployment failed: ${e.message}"
                    // Get additional debugging information
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                            echo "=== Kubernetes Configuration ==="
                            kubectl config view --minify
                            
                            echo "\\n=== Cluster Status ==="
                            kubectl cluster-info
                            
                            echo "\\n=== Events ==="
                            kubectl get events -n ${NAMESPACE}
                            
                            echo "\\n=== Deployments ==="
                            kubectl describe deployments -n ${NAMESPACE}
                            
                            echo "\\n=== Pods ==="
                            kubectl get pods -n ${NAMESPACE}
                        '
                    """
                    error "Deployment failed: ${e.message}"
                }
            }
        }
    }
}


stage('Verify Deployment') {
    steps {
        sshagent(['k8s']) {
            script {
                try {
                    // Function to check pod readiness
                    def checkPodsReady = {
                        def podsReady = sh(
                            script: """
                                ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 \
                                'kubectl get pods -n ${NAMESPACE} -o json' | \
                                jq -r '.items[] | select(.status.phase != "Running" or ([.status.conditions[] | select(.type == "Ready")] | length == 0)) | .metadata.name'
                            """,
                            returnStdout: true
                        ).trim()
                        return podsReady.isEmpty()
                    }

                    // Comprehensive verification
                    timeout(time: 5, unit: 'MINUTES') {
                        echo "Verifying deployment status..."
                        
                        // Check services and pods status
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                                echo "=== Services Status ==="
                                kubectl get services -n ${NAMESPACE} -o wide
                                
                                echo "\\n=== Pods Status ==="
                                kubectl get pods -n ${NAMESPACE} -o wide
                                
                                echo "\\n=== Resource Usage ==="
                                kubectl top pods -n ${NAMESPACE}
                            '
                        """

                        // Wait for all pods to be ready
                        waitUntil {
                            return checkPodsReady()
                        }

                        // Verify endpoints
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                                echo "\\n=== Endpoints Status ==="
                                kubectl get endpoints -n ${NAMESPACE}
                            '
                        """
                    }
                } catch (Exception e) {
                    // Collect diagnostic information
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@3.91.184.171 '
                            echo "=== Debug Information ==="
                            kubectl describe pods -n ${NAMESPACE}
                            kubectl get events -n ${NAMESPACE}
                        '
                    """
                    error "Verification failed: ${e.message}"
                }
            }
        }
    }
}
    }       
    

    // post {
    //     always {
    //         script {
    //             // Clean up containers
    //             sh 'docker-compose down --volumes --remove-orphans || true'
    //         }
    //     }
    //     success {
    //         echo "Build and Deployment Successful!"
    //     }
    //     failure {
    //         script {
    //             echo "Build or Deployment Failed!"
    //             // Capture logs on failure
    //             sh 'docker-compose logs || true'
    //         }
    //     }
    //  }
}