pipeline {
    agent any

    tools {
        nodejs 'node22'
    }

    environment {
        DOCKER_IMAGE = "manoj3003/validate:${BUILD_NUMBER}"
        GIT_REPO_NAME = "validate-Login-page"
        GIT_USER_NAME = "manoj7894"
        DEPLOYMENT_FILE_PATH = "Kubernetes/deployment.yml"  // Adjust the path if necessary
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Remove All Previous Docker Images') {
            steps {
                // Remove all Docker images
                sh 'docker rmi -f $(docker images -a -q) || true'
            }
        }
        
        stage('Cleanup Old Trivy Reports') {
            steps {
                script {
                    echo "Cleaning up old Trivy reports..."
                    sh '''
                        rm -f trivy.txt
                        rm -f fs-report.html
                    '''
                }
            }
        }
        
        stage('Get Code') {
            steps {
                git branch: 'main', url: 'https://github.com/manoj7894/validate-Login-page.git'
            }
        }
        
        stage('Install Package Dependencies') {
            steps {
                sh "npm install"
            }
        }
        
        stage('Check for Tests') {
            steps {
                script {
                    def testScriptExists = sh(script: "grep -q '\"test\":' package.json", returnStatus: true) == 0
                    env.TEST_SCRIPT_EXISTS = testScriptExists ? 'true' : 'false'
                }
            }
        }
        
        /*
        stage('Unit Test') {
            when {
                expression { env.TEST_SCRIPT_EXISTS == 'true' }
            }
            steps {
                script {
                    try {
                        sh "npm test"
                        currentBuild.result = 'SUCCESS'
                    } catch (Exception e) {
                        echo "Unit tests failed. Continuing with the pipeline."
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        } */
        
        stage('OWASP Scan') {
            when {
                expression {
                    fileExists('package.json')
                }
            }
            steps {
                dependencyCheck additionalArguments: '', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Trivy Filesystem Scan') {
            steps {
                script {
                    sh "trivy fs --format table -o fs-report.html ."
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker_id', toolName: 'Docker') {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                script {
                    sh "trivy image ${DOCKER_IMAGE} > trivy.txt"
                }
            }
        }
        
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('Sonar_Install') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Validate \
                    -Dsonar.projectKey=Validate \
                    -Dsonar.host.url=http://43.204.229.126:9000'''
                }
            }
        }
        
        stage('Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker_id', toolName: 'Docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }
        
        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github_id', variable: 'GITHUB_TOKEN')]) {
                    script {
                        sh '''
                            git config user.email "manojvarmapotthuri3003@gmail.com"
                            git config user.name "Manojvarma Potthuri"
                        '''
                        
                        sh 'git checkout main'
                        sh 'cat ${DEPLOYMENT_FILE_PATH}'
                        
                        sh '''
                            sed -i "s|image: manoj3003/validate:[^[:space:]]*|image: manoj3003/validate:${BUILD_NUMBER}|g" ${DEPLOYMENT_FILE_PATH}
                        '''
                        
                        sh '''
                            if git diff --quiet; then
                                echo "No changes detected in ${DEPLOYMENT_FILE_PATH}"
                            else
                                git add ${DEPLOYMENT_FILE_PATH}
                                git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                                git push -f https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                            fi
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            emailext attachLog: true,
                subject: "Pipeline Status: ${BUILD_NUMBER}",
                body: '''<html>
                           <body>
                              <p>Build Status: ${BUILD_STATUS}</p>
                              <p>Build Number: ${BUILD_NUMBER}</p>
                              <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                           </body>
                        </html>''',
                to: 'manojvarmapotthutri@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                attachmentsPattern: 'trivy.txt',
                mimeType: 'text/html'
        }
    }
}
