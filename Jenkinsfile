pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-pwd'
        DOCKERHUB_USERNAME = 'zainebkallel'
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/boycott-app:latest"
        KUBECONFIG_CREDENTIALS = 'kubeconfig-file'
        NAMESPACE = 'boycott'
    }

    tools {
        maven 'M2_HOME'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/zaineb827/Boycott.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarServer') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=boycott-app \
                        -Dsonar.projectName=boycott-app \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Trivy Scan - Dockerfile') {
            steps {
                sh '''
                  docker run --rm \
                    -v $(pwd):/project \
                    aquasec/trivy:latest \
                    config /project/Dockerfile
                '''
            }
        }

        stage('Build & Trivy Scan - Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'DOCKERHUB_TOKEN')]) {
                        sh """
                            # Login Docker Hub
                            echo $DOCKERHUB_TOKEN | docker login -u ${DOCKERHUB_USERNAME} --password-stdin

                            # Build Docker image
                            docker build --no-cache -t ${IMAGE_NAME} .

                            # Scan Docker image avec Trivy
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:latest image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE_NAME}

                            # Push sur Docker Hub
                            docker push ${IMAGE_NAME}
                        """
                    }
                }
            }
        }

        stage('Deploy MySQL & App to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG')]) {
                        sh "kubectl --kubeconfig=$KUBECONFIG apply -n ${NAMESPACE} -f k8s/mysql-pvc.yaml"
                        sh "kubectl --kubeconfig=$KUBECONFIG apply -n ${NAMESPACE} -f k8s/mysql-deployment.yaml"
                        sh "kubectl --kubeconfig=$KUBECONFIG apply -n ${NAMESPACE} -f k8s/mysql-service.yaml"
                        sh "kubectl --kubeconfig=$KUBECONFIG apply -n ${NAMESPACE} -f k8s/app-deployment.yaml"
                        sh "kubectl --kubeconfig=$KUBECONFIG rollout restart deployment boycott-app -n ${NAMESPACE}"
                        sh "kubectl --kubeconfig=$KUBECONFIG apply -n ${NAMESPACE} -f k8s/app-service.yaml"
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG')]) {
                        sh "kubectl --kubeconfig=$KUBECONFIG get pods -n ${NAMESPACE}"
                        sh "kubectl --kubeconfig=$KUBECONFIG get svc -n ${NAMESPACE}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline terminé avec succès !"
        }
        failure {
            echo "Pipeline échoué ! Vérifier les logs."
        }
    }
}
