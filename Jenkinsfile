pipeline {
    agent any
    
    parameters {
        string(name: 'BUILD_NUMBER', description: 'Build number from CI pipeline')
        string(name: 'DOCKER_REGISTRY', description: 'Docker registry URL')
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'emart-cluster'
        ARGOCD_SERVER = 'devops'
        ARGOCD_NAMESPACE = 'argocd'
        GIT_REPO = 'tien22521469/gitops'
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
                            aws sts get-caller-identity || { echo "AWS credentials are invalid"; exit 1; }
                            
                            # Update kubeconfig with proper error handling
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} || { echo "Failed to update kubeconfig"; exit 1; }
                            
                            # Verify cluster access
                            kubectl cluster-info || { echo "Failed to access cluster"; exit 1; }
                            
                            # Wait for ArgoCD application to be healthy
                            kubectl wait --for=condition=healthy --timeout=300s application/emartapp -n ${ARGOCD_NAMESPACE} || { echo "ArgoCD application not healthy"; exit 1; }
                            
                            # Wait for all pods to be ready
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=javaapi -n emartapp || { echo "JavaAPI pods not ready"; exit 1; }
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=nodeapi -n emartapp || { echo "NodeAPI pods not ready"; exit 1; }
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=frontend -n emartapp || { echo "Frontend pods not ready"; exit 1; }
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            
                            # Verify deployments
                            kubectl rollout status deployment/javaapi -n emartapp
                            kubectl rollout status deployment/nodeapi -n emartapp
                            kubectl rollout status deployment/frontend -n emartapp
                            
                            # Verify services
                            kubectl get svc -n emartapp
                            
                            # Verify network policies
                            kubectl get networkpolicy -n emartapp
                            
                            # Verify pod security contexts
                            kubectl get pods -n emartapp -o json | jq '.items[].spec.securityContext'
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
                    Check console output at \${env.BUILD_URL}
                    
                    Deployment Status:
                    - JavaAPI: \$(kubectl get deployment javaapi -n emartapp -o jsonpath='{.status.availableReplicas}')/\$(kubectl get deployment javaapi -n emartapp -o jsonpath='{.status.replicas}')
                    - NodeAPI: \$(kubectl get deployment nodeapi -n emartapp -o jsonpath='{.status.availableReplicas}')/\$(kubectl get deployment nodeapi -n emartapp -o jsonpath='{.status.replicas}')
                    - Frontend: \$(kubectl get deployment frontend -n emartapp -o jsonpath='{.status.availableReplicas}')/\$(kubectl get deployment frontend -n emartapp -o jsonpath='{.status.replicas}')
                """.stripIndent(),
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
        failure {
            emailext (
                subject: "CD Pipeline Failed: ${currentBuild.fullDisplayName}",
                body: """\
                    Your CD pipeline has failed.
                    Build Number: ${BUILD_NUMBER}
                    Check console output at \${env.BUILD_URL}
                    
                    Last 10 lines of logs:
                    \$(kubectl logs -n emartapp -l app=javaapi --tail=10)
                    \$(kubectl logs -n emartapp -l app=nodeapi --tail=10)
                    \$(kubectl logs -n emartapp -l app=frontend --tail=10)
                """.stripIndent(),
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
    }
}
