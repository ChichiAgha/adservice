pipeline {
    agent { label 'ubuntu' }

    environment {
        SERVICE_NAME = "adservice"
        DOCKER_REPO = "jothamcloud" 
        DOCKER_IMAGE = "${DOCKER_REPO}/${SERVICE_NAME}"
        VERSION = "${BUILD_NUMBER}"
        SONARQUBE_URL = "http://75.101.245.14:9000"
        GITEA_URL = "gitea.ajotham.link"
        MANIFESTS_REPO = "k8-manifests"
        DEPLOYMENT_FILENAME = "adservice.yml"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

      stage('SonarQube Analysis') {
    steps {
        withVault(configuration: [timeout: 60, vaultCredentialId: 'vault-approle', engineVersion: 2, vaultUrl: 'https://vault.ajotham.link'],
                vaultSecrets: [[path: 'secret/sonarqube', secretValues: [[envVar: 'SONAR_TOKEN', vaultKey: 'token']]]]) {
            script {
                sh """
                    cd src/adservice
                    chmod +x ./gradlew
                    ./gradlew sonarqube || true \
                        --init-script /root/.gradle-init/init.gradle \
                        --no-daemon \
                        -Dsonar.projectKey=${SERVICE_NAME} \
                        -Dsonar.projectName=${SERVICE_NAME} \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.java.source=17 \
                        -Dorg.gradle.daemon=false
                """
            }
        }
    }
}

        stage('Build Image') {
            steps {
                script {
                    sh """
                        echo "Building image: ${DOCKER_IMAGE}:${VERSION}"
                        docker build -t ${DOCKER_IMAGE}:${VERSION} -f src/adservice/Dockerfile src/adservice
                    """
                }
            }
        }

        stage('Initialize Trivy DB') {
            steps {
                sh """
                    echo "Initializing Trivy DB..."
                    trivy --cache-dir /var/lib/trivy image --download-db-only
                    sleep 5
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                    echo "Scanning image: ${DOCKER_IMAGE}:${VERSION}"
                    trivy image \
                        --cache-dir /var/lib/trivy \
                        ${DOCKER_IMAGE}:${VERSION} \
                        --severity HIGH,CRITICAL \
                        --exit-code 0
                """
            }
        }

        stage('Push Image') {
            steps {
                withVault(configuration: [timeout: 60, vaultCredentialId: 'vault-approle', engineVersion: 2, vaultUrl: 'https://vault.ajotham.link'],
                        vaultSecrets: [[path: 'secret/docker', 
                                      secretValues: [
                                          [envVar: 'DOCKER_USERNAME', vaultKey: 'username'],
                                          [envVar: 'DOCKER_TOKEN', vaultKey: 'token']
                                      ]]]) {
                    script {
                        sh """
                            echo "Logging into Docker Hub..."
                            echo "\${DOCKER_TOKEN}" | docker login -u "\${DOCKER_USERNAME}" --password-stdin
                            
                            if [ \$? -eq 0 ]; then
                                echo "Pushing image: ${DOCKER_IMAGE}:${VERSION}"
                                docker push ${DOCKER_IMAGE}:${VERSION}
                                
                                echo "Tagging and pushing latest..."
                                docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                                docker push ${DOCKER_IMAGE}:latest
                            else
                                echo "Docker login failed"
                                exit 1
                            fi
                        """
                    }
                }
            }
        }

        stage('Update Manifest and Create Pull Request') {
            steps {
                script {
                    withVault(configuration: [timeout: 60, vaultCredentialId: 'vault-approle', engineVersion: 2, vaultUrl: 'https://vault.ajotham.link'],
                            vaultSecrets: [[path: 'secret/gitea', 
                                        secretValues: [
                                            [envVar: 'GITEA_USERNAME', vaultKey: 'username'],
                                            [envVar: 'GITEA_PASSWORD', vaultKey: 'password']
                                        ]]]) {
                        
                        // Checkout the repository
                        checkout([$class: 'GitSCM',
                                branches: [[name: "main"]],
                                userRemoteConfigs: [[url: "https://gitea.ajotham.link/Jotham/micro-services-demo.git"]]])
                        
                        // Generate a unique branch name
                        def branchName = "update-adservice-image-v${VERSION}"
                        
                        // Create and switch to new branch
                        sh "git checkout -b ${branchName}"
                        
                        // Update the image using yq or sed
                        sh """
                                sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${VERSION}|" k8-manifests/${DEPLOYMENT_FILENAME}
                        """
                        
                        // Configure git
                        sh """
                            git config --global user.email "jenkins@ajotham.link"
                            git config --global user.name "Jenkins"
                        """
                        
                        // Stage and commit changes
                        sh """
                            git add k8-manifests/${DEPLOYMENT_FILENAME}
                            git commit -m "feat(adservice): Update image to v${VERSION}"
                        """
                        
                        // Push the new branch
                        sh """
                            # URL encode the username and password
                            USERNAME_ENCODED=\$(printf "%s" "\${GITEA_USERNAME}" | jq -sRr @uri)
                            PASSWORD_ENCODED=\$(printf "%s" "\${GITEA_PASSWORD}" | jq -sRr @uri)
                            
                            # Construct the URL with encoded credentials
                            REPO_URL="https://\${USERNAME_ENCODED}:\${PASSWORD_ENCODED}@gitea.ajotham.link/Jotham/micro-services-demo.git"
                            
                            # Set the remote URL and push
                            git remote set-url origin "\${REPO_URL}"
                            git push --set-upstream origin ${branchName}
                        """
                        
                        def requestBody = """
                            {
                                "title": "Update adservice image to v${VERSION}",
                                "body": "Automated PR: Updated adservice deployment to use image version ${VERSION}\\n\\nChanges made:\\n- Updated adservice image to ${DOCKER_IMAGE}:${VERSION}",
                                "head": "${branchName}",
                                "base": "main"
                            }
                        """
                        
                        // Using single quotes for curl command to prevent groovy interpolation of sensitive data
                        sh '''
                            curl -X POST \
                                -H "Content-Type: application/json" \
                                -u "${GITEA_USERNAME}:${GITEA_PASSWORD}" \
                                -d \'''' + requestBody + '''\' \
                                "https://gitea.ajotham.link/api/v1/repos/Jotham/micro-services-demo/pulls"
                        '''
                    }
                }
            }
        }

    }

    post {
        always {
            sh """
                echo "Cleaning up..."
                docker logout || true
                docker rmi ${DOCKER_IMAGE}:${VERSION} || true
                docker rmi ${DOCKER_IMAGE}:latest || true
                rm -rf ${WORKSPACE}/.gradle-home
                rm -rf ${WORKSPACE}/.gradle-init
            """
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
            echo "Image pushed as ${DOCKER_IMAGE}:${VERSION} and ${DOCKER_IMAGE}:latest"
        }
        failure {
            echo "Pipeline failed! Check the logs for details."
        }
    }
}