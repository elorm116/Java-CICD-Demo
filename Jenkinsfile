#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    environment {
        IMAGE_VERSION = ''
    }
    stages {
        stage('Increment Version') {
            steps {
                script {
                    echo 'Incrementing application version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    
                    // Debug: Show what grep finds
                    sh 'echo "=== First version tag in pom.xml ==="'
                    sh 'grep -m1 "<version>" pom.xml'
                    
                    // Try simplest possible extraction
                    sh 'grep -m1 "<version>" pom.xml | cut -d">" -f2 | cut -d"<" -f1 > version.txt'
                    
                    // Debug: Show file contents
                    sh 'echo "=== Content of version.txt ==="'
                    sh 'cat version.txt'
                    sh 'echo "=== End of version.txt ==="'
                    
                    env.IMAGE_VERSION = readFile('version.txt').trim()
                    echo "✅ Set IMAGE_VERSION to: ${env.IMAGE_VERSION}"
                    
                    // Validate version format
                    if (!env.IMAGE_VERSION || !(env.IMAGE_VERSION ==~ /\d+\.\d+\.\d+/)) {
                        error("❌ Invalid version extracted: ${env.IMAGE_VERSION}")
                    }
                }
            }
        }
        stage('Build App') {
            steps {
                script {
                    echo "Building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-nexus-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t host.docker.internal:8083/my-app:${env.IMAGE_VERSION} ."
                        sh "echo \$PASS | docker login host.docker.internal:8083 -u \$USER --password-stdin"
                        sh "docker push host.docker.internal:8083/my-app:${env.IMAGE_VERSION}"
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    echo 'Deploying Docker image to EC2...'
                }
            }
        }
        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-integration', variable: 'GITHUB_TOKEN')]) {
                        sh 'git config user.email "jenkins@ci.com"'
                        sh 'git config user.name "Jenkins CI"'
                        sh "git remote set-url origin https://elorm116:\${GITHUB_TOKEN}@github.com/elorm116/java-cicd-demo.git"
                        sh 'git add pom.xml'
                        sh """
                            if git diff --cached --quiet; then
                                echo "No changes to commit"
                            else
                                git commit -m "ci: version bump to ${env.IMAGE_VERSION} [skip ci]"
                                git push origin HEAD:main
                            fi
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            sh 'rm -f version.txt'
        }
        success {
            echo "✅ Pipeline completed successfully!"
            echo "🚀 Built and pushed: my-app:${env.IMAGE_VERSION}"
        }
    }
}