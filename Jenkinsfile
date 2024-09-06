pipeline {
    agent any
    tools {
        jdk 'JDK17' // Ensure the name matches the one configured in Jenkins
        maven 'maven3' // Similarly, ensure Maven is configured as 'maven3'
    }
    
    environment {
        DOCKER_CREDENTIALS_ID = 'DockerHub-Cred'
        GITHUB_CREDENTIALS = 'Git-cred'
        SONARQUBE_SERVER = 'SonarQubeServer'
        DOCKERHUB_REPO = 'ziyadtarek99/myreact-app'
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner'
        SONARQUBE_TOKEN = 'your-sonarqube-token'
        K8S_CRED_ID = 'myminikube-cred'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: "${GITHUB_CREDENTIALS}", url: 'https://github.com/ziyad-tarek1/myreact-app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('react-app') { 
                    // Install npm dependencies
                    sh "npm install"
                }
            }
        }

        stage('Compile the source code') {
            steps {
                dir('react-app') { 
                    // Compile Maven source code
                    sh "mvn compile"
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                dir('react-app') { 
                    // Run Maven unit tests
                    sh "mvn test"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    // Run SonarQube scan
                    sh "${SONAR_SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=react-app -Dsonar.sources=. -Dsonar.host.url=http://192.168.209.135:9000 -Dsonar.login=${SONARQUBE_TOKEN}"
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                // Run a Trivy file system scan
                sh 'docker run --rm -v $(pwd):/app aquasec/trivy:latest fs /app'
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('react-app') { 
                    script {
                        // Build the Docker image and tag it using the build number
                        def newTag = "${env.BUILD_NUMBER}.0"
                        env.IMAGE_TAG = newTag
                        
                        docker.build("${DOCKERHUB_REPO}:${newTag}")
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // Push Docker image to DockerHub
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        docker.image("${DOCKERHUB_REPO}:${IMAGE_TAG}").push()
                        docker.image("${DOCKERHUB_REPO}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                // Run a Trivy image scan for vulnerabilities
                sh "docker run --rm aquasec/trivy:latest image ${DOCKERHUB_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Update Deployment File and Push to Test Branch') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIALS}", usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        // Switch to a new test branch, commit and push changes
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
            cleanWs() // Removed node block, which caused the issue
        }
    }
}
