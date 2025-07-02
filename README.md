install java, docker, trivy, kubectl on jenkins
docker run -d -p 8081:8081 sonatype/nexus3
docker run -d -p 9000:9000 sonarqube:lts-community 
sudo usermod -aG docker ${USER}
newgrp docker
sudo snap install kubectl
aws eks update-kubeconfig --name eks-cluster --region us-east-1
enabel rbac - create sa, role, role bindind, create token and add in jenkins
add token from sonarqube on jenkins(security->users->token on sonarqube dashboard)
configure sonarqube server using this credentials
configure manage nodes with nexus username and pass
add the nexus url in pom.xml


Issue faced
docker build failed as the docker grp access for jenkins was given but jenkins was not restarted


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
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-credentials', url: 'https://github.com/bibinpanicker07/Java-project-1.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Tests') {
            steps {
                sh "mvn clean test -X -DskipTests=true"
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
                sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=BlueGreen -Dsonar.projectName=BlueGreen -Dsonar.java.binaries=target"
                }
                
            }
        }
        
        
        stage('Code Quality check') {
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
                    withDockerRegistry(credentialsId: 'docker-credentials') {
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
                    withDockerRegistry(credentialsId: 'docker-credentials') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4BEB157C2D0297DBF0C5626430EF198.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE} --validate=false"
                        
                    }
                    
                }
                
            }
            
        }
        
        
        
        stage('Deploy SVC app') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4BEB157C2D0297DBF0C5626430EF198.gr7.us-east-1.eks.amazonaws.com') {
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
                    
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4BEB157C2D0297DBF0C5626430EF198.gr7.us-east-1.eks.amazonaws.com') {
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
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4BEB157C2D0297DBF0C5626430EF198.gr7.us-east-1.eks.amazonaws.com') {
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
                    withKubeConfig(caCertificate: '', clusterName: 'eks-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4BEB157C2D0297DBF0C5626430EF198.gr7.us-east-1.eks.amazonaws.com') {
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
