pipeline {
    agent any

    stages {
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
                        withCredentials([usernamePassword(credentialsId: 'github-integration', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                            sh """
                                ssh -o StrictHostKeyChecking=no -T ec2-user@34.239.177.1 << 'EOF'
echo "\$PASS" | docker login ghcr.io -u "\$USER" --password-stdin
docker pull ghcr.io/elorm116/my-app:v2
docker stop my-app || true
docker rm my-app || true
docker run -d --name my-app -p 3000:80 ghcr.io/elorm116/my-app:v2
EOF
                            """
                        }
                    }
                }
            }
        }
    }
}