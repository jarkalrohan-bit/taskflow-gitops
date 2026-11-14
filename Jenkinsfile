pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag built by CI')
        string(name: 'DOCKERHUB_USER', defaultValue: 'rohanjarkal', description: 'Docker Hub image name')
        string(name: 'BACKEND_IMAGE', defaultValue: 'taskflow-backend')
        string(name: 'FRONTEND_IMAGE', defaultValue: 'taskflow-frontend')
    }

    environment {
        MANIFEST_REPO = 'https://github.com/jarkalrohan-bit/taskflow-gitops.git'
        BACKEND_IMAGE = "taskflow-backend"
        FRONTEND_IMAGE = "taskflow-frontend"
    }

    stages {
        stage('Checkout GitOps Repository') {
            steps {
                script {
                    git(
                        url: "https://github.com/jarkalrohan-bit/taskflow-gitops.git", branch: "main" 
                        )
                }
            }
        }

        stage('Update Image Tags') {
            steps {
                script {
                        // 1. Update Frontend Image
                        sh """
                            echo "Updating Frontend image..."
                            # Using sed (replace with yq if available for safer parsing)
                            sed -i 's|image:.*|image: ${params.DOCKERHUB_USER}/${params.FRONTEND_IMAGE}:${params.IMAGE_TAG}|g' frontend/deployment.yaml
                        """
        
                        // 2. Update Backend Image
                        sh """
                            echo "Updating Backend image..."
                            sed -i 's|image:.*|image: ${params.DOCKERHUB_USER}/${params.BACKEND_IMAGE}:${params.IMAGE_TAG}|g' backend/deployment.yaml
                        """
        
                        // 3. Verify the changes
                        sh '''
                            echo "--- Frontend Diff ---"
                            git diff frontend/deployment.yaml
                            echo "--- Backend Diff ---"
                            git diff backend/deployment.yaml
                        '''
                }
            }
        }
 
        stage('Commit and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "gitCred",
                    passwordVariable: "GIT_PASS",
                    usernameVariable: "GIT_USER"
                )]) {
                    sh """
                        # 1. Set Git User for the Commit
                        git config user.email "jarkalrohan@gmail.com"
                        git config user.name "jarkalrohan-bit"
        
                        # 2. Add and Commit
                        git add frontend/deployment.yaml backend/deployment.yaml
                        
                        # Only commit if there are actual changes to prevent pipeline failure on empty commits
                        if git diff --cached --quiet; then
                            echo "No changes to commit. Skipping push."
                        else
                        # 3. Securely Push using a temporary URL (Password is NOT saved to config)
                        # We pass the URL directly to the 'git push' command.
                            git commit -m "CD: Update image tags to ${params.IMAGE_TAG} [skip ci]"
                            git push "https://github.com/jarkalrohan-bit/taskflow-gitops.git" HEAD:main
                        fi
                    
                    """
                }
            }
        }
        
    }
 
    post {
        always {
            script {
                def color = (currentBuild.currentResult == 'SUCCESS') ? '#2ecc71' : (currentBuild.currentResult == 'UNSTABLE' ? '#f1c40f' : '#e74c3c')
                def statusText = currentBuild.currentResult
                def duration = "${currentBuild.durationString.replace(' and counting', '')}"
                
                // Optional: get who triggered the build
                def triggerCause = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userName ?: 'SCM / Timer'

                emailext(
                    attachLog: (currentBuild.currentResult != 'SUCCESS'),
                    from: 'jarkalrohan@gmail.com',
                    to: 'jarkalrohan@gmail.com',
                    replyTo: 'myemail@gmail.com',
                    subject: "${statusText}: ${env.JOB_NAME} [Build #${env.BUILD_NUMBER}]",
                    mimeType: 'text/html',
                    body: """
                        <html>
                        <body style="font-family: Arial, sans-serif; color: #333;">
                            <div style="border-left: 6px solid ${color}; padding-left: 16px; margin-bottom: 20px;">
                                <h2 style="color: ${color}; margin: 0;">Build ${statusText}</h2>
                                <p style="margin: 4px 0;"><strong>Job:</strong> ${env.JOB_NAME}</p>
                                <p style="margin: 4px 0;"><strong>Build Number:</strong> #${env.BUILD_NUMBER}</p>
                                <p style="margin: 4px 0;"><strong>Duration:</strong> ${duration}</p>
                                <p style="margin: 4px 0;"><strong>Triggered by:</strong> ${triggerCause}</p>
                            </div>

                            <p><strong>Console Output:</strong><br/>
                            <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>

                            <p><strong>Blue Ocean:</strong><br/>
                            <a href="${env.BUILD_URL}blue">View in Blue Ocean</a></p>

                            <p style="font-size: 12px; color: #888; margin-top: 30px;">
                                Sent automatically by Jenkins CI/CD · ${env.JOB_URL}
                            </p>
                        </body>
                        </html>
                    """
                )
            }
        }
    }
}  