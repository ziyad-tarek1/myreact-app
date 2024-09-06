pipeline {
    agent any // Runs on any available Jenkins agent
    tools {
        jdk 'JDK17' // Make sure 'JDK17' is configured in Jenkins (Manage Jenkins -> Global Tool Configuration)
        maven 'maven3' // Ensure Maven is set as 'maven3' in Jenkins (Manage Jenkins -> Global Tool Configuration)
    }
    
    environment {
        DOCKER_CREDENTIALS_ID = 'DockerHub-Cred' // Credentials ID for DockerHub stored in Jenkins Credentials
        GITHUB_CREDENTIALS = 'Git-cred' // GitHub credentials ID stored in Jenkins Credentials
        SONARQUBE_SERVER = 'SonarQubeServer' // Name of the SonarQube server configuration in Jenkins
        DOCKERHUB_REPO = 'ziyadtarek99/myreact-app' // DockerHub repository for your image
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner' // Reference to SonarQube Scanner tool in Jenkins
        SONARQUBE_TOKEN = 'SonarQube'  // this is my sonarqube secret text (token) credintial ID  // Your SonarQube authentication token, needs to be configured
        K8S_CRED_ID = 'myminikube-cred' // Kubernetes credentials stored in Jenkins
    }

    stages {
        stage('Clean Workspace') {
            steps {
                echo 'Cleaning workspace...' // Debug message
                cleanWs() // Removes any files from the workspace before the pipeline starts
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out the code from GitHub...' // Debug message
                git branch: 'main', credentialsId: "${GITHUB_CREDENTIALS}", url: 'https://github.com/ziyad-tarek1/myreact-app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing npm dependencies...' // Debug message
                dir('react-app') { 
                    sh "npm install" // Install npm dependencies inside the react-app directory
                }
            }
        }

        stage('Compile the source code') {
            steps {
                echo 'Compiling the source code...' // Debug message
                dir('react-app') { 
                    sh "mvn compile" // Compiling Java code using Maven
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...' // Debug message
                dir('react-app') { 
                    sh "mvn test" // Running unit tests with Maven
                }
            }
        }

withCredentials([string(credentialsId: 'SonarQube', variable: 'SONARQUBE_TOKEN')]) {
    stage('SonarQube Analysis') {
        steps {
            echo 'Starting SonarQube analysis...' // Debug message
            withSonarQubeEnv(installationName: 'SonarQubeServer') {
                // Print variables for debugging
                sh "echo SONAR_SCANNER_HOME = ${SONAR_SCANNER_HOME}"
                sh "echo SONARQUBE_TOKEN = ${SONARQUBE_TOKEN}"
                sh "echo SONAR_HOST_URL = http://192.168.209.135:9000" // Update to your actual SonarQube URL

                // Run SonarQube analysis on the project
                sh "${SONAR_SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=react-app -Dsonar.sources=. -Dsonar.host.url=http://192.168.209.135:9000 -Dsonar.login=${SONARQUBE_TOKEN}"
            }
        }
    }
}
        stage('Trivy FS Scan') {
            steps {
                echo 'Running Trivy file system scan...' // Debug message
                sh 'docker run --rm -v $(pwd):/app aquasec/trivy:latest fs /app' // Scans file system for vulnerabilities using Trivy
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...' // Debug message
                dir('react-app') {
                    script {
                        def newTag = "${env.BUILD_NUMBER}.0" // Tag Docker image with the current Jenkins build number
                        env.IMAGE_TAG = newTag
                        
                        docker.build("${DOCKERHUB_REPO}:${newTag}") // Build the Docker image with the generated tag
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to DockerHub...' // Debug message
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        // Push the Docker image with both the build-specific tag and 'latest' tag
                        docker.image("${DOCKERHUB_REPO}:${IMAGE_TAG}").push()
                        docker.image("${DOCKERHUB_REPO}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                echo 'Running Trivy Docker image scan...' // Debug message
                // Run Trivy vulnerability scan on the built Docker image
                sh "docker run --rm aquasec/trivy:latest image ${DOCKERHUB_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...' // Debug message
                // Wait for SonarQube quality gate result (whether the code passed quality standards)
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true // Fails the pipeline if quality gate is not met
                }
            }
        }

        stage('Update Deployment File and Push to Test Branch') {
            steps {
                echo 'Updating deployment files and pushing to test branch...' // Debug message
                script {
                    withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIALS}", usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        // Checkout or create the 'test' branch, add changes, commit, and push them to GitHub
                        sh 'git checkout -b test || git checkout test'
                        sh 'git add .'
                        sh "git commit -m 'Update deployment files and test cases'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/ziyad-tarek1/myreact-app.git HEAD:test"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace after job completion...' // Debug message
            cleanWs() // Clean up the workspace after the pipeline completes
        }
    }
}
