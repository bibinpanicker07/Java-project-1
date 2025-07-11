pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "bibin2025/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'sonar-scanner'
        EKS_SERVER_URL = ''
    }
    

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/bibinpanicker07/Java-project-1.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Tests') {
            steps {
                sh "mvn clean test"
            }
        }
        
        stage('Trivy FS scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        
        stage('Sonarqube analysis') {
            steps {
            withSonarQubeEnv('sonar') {
                sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=JavaApp -Dsonar.projectName=JavaApp -Dsonar.java.binaries=target"
                }
                
            }
        }
        
        
        stage('Quality gate check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                    
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Publish artifact To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -X -DskipTests=true"
                }
            }
        }
        
        stage('Docker Build & tag image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
        
         stage('trivy image scan') {
            steps {
                sh "trivy image --format table -o fs.html ${IMAGE_NAME}:${TAG}"
            }
        }
        
        stage('Docker Push image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: "${KUBE_NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"
                        
                    }
                    
                }
                
            }
            
        }
        
        
        
        stage('Deploy SVC app') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: "${KUBE_NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                        }
                    }
                }
            }
        
        
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }
                    
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: "${KUBE_NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                        
                    }
                }
            }
        }
        
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: "${KUBE_NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: "${KUBE_NAMESPACE}", restrictKubeConfigAccess: false, serverUrl: "${EKS_SERVER_URL}") {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }     
    }
}
