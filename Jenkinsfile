pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "ishanpathak98/ai-chatbot:latest"
        GITHUB_REPO = "https://github.com/ishanpathak98/ai-chatbot.git"
        DEPLOY_SERVER = "ec2-user@54.147.63.142"
    }

    stages {
        stage('Checkout') {
            steps {
                // ✅ No credentials needed for public GitHub repo
                git url: "${env.GITHUB_REPO}", branch: 'main'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $DOCKER_IMAGE .'
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                // ✅ Allow Trivy to fail without breaking the build
                sh 'trivy image $DOCKER_IMAGE || true'
            }
        }

        stage('Push to DockerHub') {
            steps {
                // ✅ Make sure you have 'dockerhub_creds' configured in Jenkins
                withCredentials([usernamePassword(credentialsId: 'dockerhub_creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                // ✅ Make sure 'ec2_ssh_key' is configured in Jenkins credentials
                sshagent(['ec2_ssh_key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER '
                        docker pull $DOCKER_IMAGE &&
                        docker stop chatbot || true &&
                        docker rm chatbot || true &&
                        docker run -d --name chatbot -p 3000:3000 $DOCKER_IMAGE
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${DEPLOY_SERVER.split('@')[1]}:3000", returnStdout: true).trim()
                    if (response != '200') {
                        error "❌ Health check failed! App is not responding properly."
                    } else {
                        echo "✅ Health check passed!"
                    }
                }
            }
        }
    }

    post {
        success {
            mail to: 'ishaanpathak94@gmail.com',
                 subject: "✅ SUCCESS: Jenkins Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "The CI/CD pipeline executed successfully. Docker image: $DOCKER_IMAGE"
        }
        failure {
            mail to: 'ishaanpathak94@gmail.com',
                 subject: "❌ FAILURE: Jenkins Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "The CI/CD pipeline failed. Please check Jenkins logs."
        }
    }
}
