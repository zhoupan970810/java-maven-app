// #!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }

    environment {
        // Application Configuration
        APP_NAME = 'java-maven-app'
        DOCKER_REPO = 'zhoupan970810/java-maven-app'
        KUBECONFIG = '/root/.kube/config'

        // Git Configuration
        GIT_USER_EMAIL = 'jenkins@automation.local'
        GIT_USER_NAME = 'Jenkins CI/CD'

        // EKS Configuration - Update these or use Jenkins parameters
        AWS_REGION = 'eu-central-1'
        EKS_CLUSTER_NAME = 'eks-cluster'
    }

    stages {

        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_TAG = "$version-$BUILD_NUMBER"
                }
            }
        }

        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }

        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t ${DOCKER_REPO}:${IMAGE_TAG} ."
                        sh '''
                            echo ${PASS} | docker login -u ${USER} --password-stdin
                        '''
                        sh "docker push ${DOCKER_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
            }
            steps {
                script {
                    echo 'deploying docker image to EKS...'

                    // Ensure kubectl can connect to EKS
                    sh '''
                        # Test EKS connection
                        echo "Testing EKS connection..."
                        kubectl cluster-info || echo "Cluster info check - continuing"
                        kubectl get nodes || echo "Node list check - continuing"

                        # Apply Kubernetes manifests
                        echo "Deploying to EKS..."
                        envsubst < kubernetes/deployment.yaml | kubectl apply -f -
                        envsubst < kubernetes/service.yaml | kubectl apply -f -

                        # Wait for deployment to be ready
                        echo "Waiting for deployment to be ready..."
                        kubectl rollout status deployment/${APP_NAME} ==timeout=5m || echo "Rollout check - continuing..."

                        # Show deployment status
                        kubectl get pods
                        kubectl get services
                    '''
                }
            }
        }

        stage('commit version update'){
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh '''
                            # Configure Git user (required for commit)
                            git config --global user.email "jenkins@automation.local"
                            git config --global user.name "Jenkins CI/CD"

                            # Set remote URL with credentials
                            git remote set-url origin https://${USER}:${PASS}@github.com/zhoupan970810/java-maven-app.git

                            # Check if there are changes to commit
                            if git status --porcelain | grep -q .; then
                                echo "Changes detected, committing..."
                                git add .
                                git commit -m "ci: version bump to ${IMAGE_TAG} [skip ci]"
                                echo "Pushing changes..."
                                git push origin HEAD:jenkins-jobs
                            else
                                echo "No changes to commit."
                            fi
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
            script {
                // Clean up Docker images
                sh '''
                  echo "Cleaning up old Docker images..."
                  docker image prune -f
                '''

                // Send success notification (customize as needed)
                echo "Deployment ${IMAGE_TAG} is live on EKS!"
            }
        }
        failure {
            echo '❌ Pipeline failed!'
            script {
                // Rollback if deployment failed
                if (env.STAGE_NAME == 'deploy') {
                    echo 'Deployment failed! Rolling back...'
                    sh '''
                        kubectl rollout undo deployment/${APP_NAME} || echo "Rollback check - continuing..."
                    '''
                }

                // Clean up failed build
                echo "Cleaning up failed build..."
                sh "docker rmi ${DOCKER_REPO}:${IMAGE_TAG} || true"
            }
        }
        always {
            // Always clean up workspace
            cleanWs()
            echo 'Pipeline execution finished.'
        }
    }
}
