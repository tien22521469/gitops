pipeline {
    agent any
    
    parameters {
        string(name: 'BUILD_NUMBER', description: 'Build number from CI pipeline')
        string(name: 'DOCKER_REGISTRY', description: 'Docker registry URL')
        string(name: 'IMAGE_TAG', description: 'Image tag to deploy')
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'emart-cluster'
        ARGOCD_SERVER = 'devops'
        ARGOCD_NAMESPACE = 'argocd'
        GIT_REPO = 'tien22521469/gitops'
        EKS_ROLE_ARN = 'arn:aws:iam::449663538285:role/EKSClusterRole'
    }
    
    stages {
        stage('Checkout GitOps Repository') {
            steps {
                cleanWs()
                git branch: 'main', credentialsId: 'github', url: "https://github.com/${GIT_REPO}.git"
            }
        }
        
        stage('Verify Kustomization') {
            steps {
                script {
                    sh """
                        cd k8s/overlays/development
                        kustomize build .
                    """
                }
            }
        }
        
        stage('Wait for ArgoCD Sync') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            # Verify AWS credentials
                            aws sts get-caller-identity

                            # Update kubeconfig
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                        """
                    }
                }
            }
        }
        
        stage('Update Deployment Images') {
            steps {
                sh '''
                    # Cập nhật image cho frontend
                    sed -i "s|nguyentienuit/emartapp-frontend:[0-9A-Za-z._-]*|nguyentienuit/emartapp-frontend:${IMAGE_TAG}|g" k8s/base/frontend-deployment.yaml

                    # Cập nhật image cho javaapi
                    sed -i "s|nguyentienuit/emartapp-javaapi:[0-9A-Za-z._-]*|nguyentienuit/emartapp-javaapi:${IMAGE_TAG}|g" k8s/base/backend-deployment.yaml

                    # Cập nhật image cho nodeapi
                    sed -i "s|nguyentienuit/emartapp-nodeapi:[0-9A-Za-z._-]*|nguyentienuit/emartapp-nodeapi:${IMAGE_TAG}|g" k8s/base/backend-deployment.yaml

                    # Cập nhật image cho postgres (nếu cần)
                    # sed -i "s|postgres:[0-9A-Za-z._-]*|postgres:${IMAGE_TAG}|g" k8s/base/database-deployment.yaml

                    # Kiểm tra lại kết quả
                    cat k8s/base/frontend-deployment.yaml
                    cat k8s/base/backend-deployment.yaml
                    cat k8s/base/database-deployment.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            
                            # Verify deployments
                            #kubectl rollout status deployment/javaapi -n emartapp
                            #kubectl rollout status deployment/nodeapi -n emartapp
                            #kubectl rollout status deployment/frontend -n emartapp
                            
                            # Verify services
                           # kubectl get svc -n emartapp
                            
                            # Verify network policies
                            #kubectl get networkpolicy -n emartapp
                            
                            # Verify pod security contexts
                            #kubectl get pods -n emartapp -o json | jq '.items[].spec.securityContext'
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            emailext (
                subject: "CD Pipeline Successful: ${currentBuild.fullDisplayName}",
                body: """\
                    Your CD pipeline has completed successfully.
                    Build Number: ${BUILD_NUMBER}
                    Check console output at ${env.BUILD_URL}
                    
                    Deployment Status (check console log for details).
                """.stripIndent(),
                to: 'your-email@example.com'
            )
        }
        failure {
            emailext (
                subject: "CD Pipeline Failed: ${currentBuild.fullDisplayName}",
                body: """\
                    Your CD pipeline has failed.
                    Build Number: ${BUILD_NUMBER}
                    Check console output at ${env.BUILD_URL}
                    
                    Please check the console log for more details and recent pod logs.
                """.stripIndent(),
                to: 'your-email@example.com'
            )
        }
    }
}
