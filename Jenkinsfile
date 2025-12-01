pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "Building project..."
            }
        }

        stage('Test') {
            steps {
                echo "Running tests..."
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sshagent(['ec2-server-key']) {
                        withCredentials([usernamePassword(credentialsId: 'docker-nexus-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                            sh """
                                ssh -o StrictHostKeyChecking=no ec2-user@34.239.177.1 '
                                    echo \$PASS | docker login 89690eacab5e.ngrok-free.app -u \$USER --password-stdin
                                    docker pull 89690eacab5e.ngrok-free.app/my-app:latest
                                    docker run -d --name my-app -p 3000:80 89690eacab5e.ngrok-free.app/my-app:latest
                                '
                            """
                        }
                    }
                }
            }
        }
    }
}
