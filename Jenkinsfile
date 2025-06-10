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
            stage('Update Deployment Images') {
            steps {
                sh '''
                    # Cập nhật image cho frontend
                    sed -i "s|nguyentienuit/emartapp-frontend:[0-9A-Za-z._-]*|nguyentienuit/emartapp-frontend:${IMAGE_TAG}|g" k8s/base/frontend-deployment.yaml

                    # Cập nhật image cho javaapi
                    sed -i "s|nguyentienuit/emartapp-javaapi:[0-9A-Za-z._-]*|nguyentienuit/emartapp-javaapi:${IMAGE_TAG}|g" k8s/base/backend-deployment.yaml

                    # Cập nhật image cho nodeapi
                    sed -i "s|nguyentienuit/emartapp-nodeapi:[0-9A-Za-z._-]*|nguyentienuit/emartapp-nodeapi:${IMAGE_TAG}|g" k8s/base/backend-deployment.yaml

                 
                    # Kiểm tra lại kết quả
                    cat k8s/base/frontend-deployment.yaml
                    cat k8s/base/backend-deployment.yaml
                    cat k8s/base/database-deployment.yaml
                '''
            }
        }
        stage("Push the changed deployment file to GitHub") {
            steps {
                sh """
                    git config --global user.name "jenkins"
                    git config --global user.email "22521469@gm.uit.edu.vn"
                    git add k8s/base/*.yaml
                    git commit -m "Updated Deployment Manifest - Image Tag ${IMAGE_TAG} (Build ${BUILD_NUMBER})" || echo "No changes to commit"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                    sh "git push https://github.com/${GIT_REPO}.git main"
                }
            }
        }
 
        stage('Wait for ArgoCD Sync') {
            steps {
                 script {
                    withCredentials([
                         [
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-credentials',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        ]
                    ]) {
                        sh """
                            # Kiểm tra AWS credentials
                            aws sts get-caller-identity
                            aws_access_key_id=${AWS_ACCESS_KEY_ID}
                            aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}
                            region=${AWS_REGION}
                            EOF

                            # Cập nhật kubeconfig với profile
                             aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --profile default

                            # Kiểm tra kết nối cluster
                            kubectl get nodes

                            # Kiểm tra trạng thái ArgoCD
                            kubectl wait --for=condition=healthy --timeout=300s application/emartapp -n ${ARGOCD_NAMESPACE}

                            # Kiểm tra trạng thái các pod
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=javaapi -n emartapp
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=nodeapi -n emartapp
                            kubectl wait --for=condition=ready --timeout=300s pod -l app=frontend -n emartapp
                        """
                        }
                    }
                }
            }
            
            stage('Verify Deployment') {
                steps {
                    script {
                        withCredentials([
                            [
                                $class: 'AmazonWebServicesCredentialsBinding',
                                credentialsId: 'aws-credentials',
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                            ]
                        ]) {
                            sh """
                                aws_access_key_id=${AWS_ACCESS_KEY_ID}
                                aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}
                                region=${AWS_REGION}
                                EOF

                                # Cập nhật kubeconfig với profile
                                aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --profile default

                                # Kiểm tra deployments
                                kubectl rollout status deployment/javaapi -n emartapp
                                kubectl rollout status deployment/nodeapi -n emartapp
                                kubectl rollout status deployment/frontend -n emartapp

                                # Kiểm tra services
                                kubectl get svc -n emartapp

                                # Kiểm tra network policies
                                kubectl get networkpolicy -n emartapp

                                # Kiểm tra pod security contexts
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
                    to: '22521469@gm.uit.edu.vn'
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
                    to: '22521469@gm.uit.edu.vn'
                )
            }
        }
    }
