pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "vishwpay/bloggingapp"   // ← Change this
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', 
                    credentialsId: 'git-cred', 
                    url: 'https://github.com/payalvishwekar/fullstack-blog-demo-app.git'   // ← VERY IMPORTANT
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=bloggingapp \
                          -Dsonar.projectKey=bloggingapp \
                          -Dsonar.java.binaries=.'''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-cred', toolName: 'docker') {
                        sh "docker build -t ${DOCKER_IMAGE}:latest ."
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE}:latest"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-cred', toolName: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8s-token', clusterName: 'my-blog-cluster') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        // stage('Verify Deployment') {
        //     steps {
        //         withKubeConfig(credentialsId: 'k8s-token', clusterName: 'my-blog-cluster') {
        //             sh "kubectl get pods -n default"
        //             sh "kubectl get svc -n default"
        //         }
        //     }
        // }
        stage('Deploy To Kubernetes') {
        steps {
        withCredentials([file(credentialsId: 'k8s-config', variable: 'KUBECONFIG')]) {
            sh '''
                cp $KUBECONFIG kubeconfig.yaml
                export KUBECONFIG=kubeconfig.yaml
                
                kubectl apply -f deployment-service.yaml
                sleep 15
                kubectl get pods
                kubectl get svc
            '''
        }
    }
}
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
    }
}
