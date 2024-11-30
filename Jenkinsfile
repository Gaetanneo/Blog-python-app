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

        // stage('Clean Environment') {
        //     steps {
        //         script {
        //             // Clean up any existing containers and volumes
        //             sh '''
        //                 docker-compose down --volumes --remove-orphans
        //                 docker system prune -f
        //             '''
        //         }
        //     }
        // }

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
        stage('Deploy to K8s Cluster') {
            steps {
                withCredentials([string(credentialsId: 'secret_token', variable: 'KUBE_TOKEN')]) {
                    sh '''
                    curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"
                    chmod u+x ./kubectl
                    export KUBECONFIG=$(mktemp)
                    ./kubectl config set-cluster kubernetes --server=https://54.90.64.161:6443 --insecure-skip-tls-verify=true
                    ./kubectl config set-credentials jenkins --token=${KUBE_TOKEN}
                    ./kubectl config set-context default --cluster=kubernetes --user=jenkins --namespace=default
                    ./kubectl config use-context default
                    ./kubectl get nodes
                    ./kubectl apply -f deployment-mysql.yml
                    ./kubectl apply -f deployment-flask.yml
                    ./kubectl apply -f flask-service.yml
                    ./kubectl apply -f mysql-service.yml
                    ./kubectl apply -f env-config.yml
                    ./kubectl apply -f persistent-volume.yml
                    ./kubectl apply -f pvc-claim.yml
                    ./kubectl apply -f sql-inject-config.yml
                    ./kubectl apply -f storage.yml
                    '''
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