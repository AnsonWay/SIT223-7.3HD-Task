pipeline {
    agent {
        docker {
            image 'maven:3.8.4-openjdk-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        SONARQUBE_URL = 'http://sonarqube:9000'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') 
        IMAGE_NAME = "ansonway/sit223-petclinic" 
        STAGING_PORT = 8082
        PROD_PORT = 8081
    }
    tools {
        maven 'M3' 
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AnsonWay/SIT223-7.3HD-Task.git'
            }
        }
        stage('Build') {
            steps {
                script {
                    echo "--- Building the application ---"
                    sh 'mvn clean package'
                    echo "--- Build successful, JAR created ---"
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }
        stage('Test') {
            steps {
                echo "--- Running automated tests ---"
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }
        stage('Code Quality Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner' 
                    withSonarQubeEnv('MySonarQube') { 
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Security Scan') {
            steps {
                echo "--- Performing Security Scans ---"
                sh 'mvn org.owasp:dependency-check-maven:8.4.0:check'

                script {
                    def customImage = docker.build(IMAGE_NAME, ".")
                }

                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 1 --severity CRITICAL ${IMAGE_NAME}:latest"
            }
        }
        stage('Deploy to Staging') {
            steps {
                echo "--- Deploying to Staging Environment ---"
                sh "docker stop staging-petclinic || true"
                sh "docker rm staging-petclinic || true"
                sh "docker run -d --name staging-petclinic -p ${STAGING_PORT}:8080 ${IMAGE_NAME}:latest"
            }
        }
        stage('Release') {
            steps {
                input message: 'Deploy to Production?', submitter: 'admin'

                script {
                    def version = sh(returnStdout: true, script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout').trim()
                    def taggedImage = "${IMAGE_NAME}:${version}-${env.BUILD_NUMBER}"
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS) {
                        docker.image(IMAGE_NAME).push(taggedImage)
                        docker.image(IMAGE_NAME).push('latest')
                    }
                }
            }
        }
        stage('Deploy to Production') {
            steps {
                echo "--- Deploying to Production Environment ---"
                sh "docker stop prod-petclinic || true"
                sh "docker rm prod-petclinic || true"
                sh "docker run -d --name prod-petclinic -p ${PROD_PORT}:8080 ${IMAGE_NAME}:latest"
            }
        }
        stage('Monitoring') {
            steps {
                echo "--- Application deployed to Production ---"
                echo "Monitoring via Prometheus at http://localhost:9090"
                echo "View dashboards in Grafana at http://localhost:3000"
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}