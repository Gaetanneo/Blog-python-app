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
        KUBE_CONFIG = "jenkins-secret"
        PROJECT_NAME = "flask-mysql"
        DOCKER_BUILDKIT = '1'
        
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

        stage('Validate Kubernetes Connection') {
    steps {
        withCredentials([file(credentialsId: 'jenkins-secret', variable: 'KUBECONFIG')]) {
            script {
                try {
                    // Check cluster connectivity
                    sh '''
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl cluster-info
                        kubectl get nodes
                        kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
                    '''
                } catch (Exception e) {
                    error "Failed to connect to Kubernetes cluster: ${e.message}"
                }
            }
        }
    }
}

stage('Deploy to Kubernetes') {
    steps {
        withCredentials([file(credentialsId: 'jenkins-secret', variable: 'KUBECONFIG')]) {
            script {
                try {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG}
                        
                        # Apply Kubernetes configurations
                        kubectl apply -f k8s/deployment-flask.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/deployment-mysql.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/flask-service.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/mysql-service.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/persistent-volume.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/pvc-claim.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/sql-inject-config.yml -n ${NAMESPACE}
                        kubectl apply -f k8s/storage.yml -n ${NAMESPACE}
                        
                        # Wait for deployments
                        kubectl rollout status deployment/flask-app -n ${NAMESPACE} --timeout=300s
                        kubectl rollout status deployment/mysql -n ${NAMESPACE} --timeout=300s
                    '''
                } catch (Exception e) {
                    error "Deployment failed: ${e.message}"
                }
            }
        }
    }
}

stage('Verify Deployment') {
    steps {
        withCredentials([file(credentialsId: 'jenkins-secret', variable: 'KUBECONFIG')]) {
            script {
                sh '''
                    export KUBECONFIG=${KUBECONFIG}
                    echo 'Checking pod status...'
                    kubectl get pods -n ${NAMESPACE}
                    echo 'Checking service status...'
                    kubectl get services -n ${NAMESPACE}
                '''
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