pipeline {
    agent any
    
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Container Version Tag')
    }
    
    environment {
        DOCKER_HUB_CREDS = 'docker-hub-credentials-id' 
        IMAGE_NAME = 'YOUR_DOCKERHUB_USERNAME/traffic-dashboard'
    }
    
    stages {
        stage('Pull Sources') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        
        stage('Parallel Automated Verification') {
            parallel {
                stage('Verify App Files') {
                    steps {
                        sh 'grep -q "Metro Traffic Control" index.html'
                    }
                }
                stage('Verify Deployment Setup') {
                    steps {
                        sh 'test -f deployment.yaml'
                    }
                }
            }
        }
        
        stage('Build & Containerize Artifacts') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${params.IMAGE_TAG} ."
            }
        }
        
        stage('Distribute Image to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_HUB_CREDS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                    sh "docker push ${IMAGE_NAME}:${params.IMAGE_TAG}"
                }
            }
        }
        
        stage('Orchestrate Deployment to Kubernetes') {
            steps {
                sh "sed -i 's|YOUR_DOCKERHUB_USERNAME/traffic-dashboard:latest|${IMAGE_NAME}:${params.IMAGE_TAG}|g' deployment.yaml"
                sh "kubectl apply -f deployment.yaml"
            }
        }
        
        stage('Rollout Verification & Automated Rollback') {
            steps {
                script {
                    try {
                        sh "kubectl rollout status deployment/traffic-management-system --timeout=90s"
                        echo "Deployment complete and healthy!"
                    } catch (Exception e) {
                        echo "⚠️ CRITICAL RECOVERY: Deployment unstable! Rolling back automatically..."
                        sh "kubectl rollout undo deployment/traffic-management-system"
                        error("Rollback completed due to failed application release check.")
                    }
                }
            }
        }
    }
}
