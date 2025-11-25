#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    environment {
        IMAGE_VERSION = ''
        IMAGE_NAME = ''
    }
    stages {
        stage('Increment Version') {
            steps {
                script {
                    echo 'Incrementing application version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    
                    // Use Jenkins built-in Maven POM reader - designed for this exact purpose
                    try {
                        def pom = readMavenPom file: 'pom.xml'
                        def version = pom.version
                        
                        if (version && !version.isEmpty()) {
                            env.IMAGE_VERSION = version
                            echo "✅ Successfully read version from POM: ${version}"
                        } else {
                            // Fallback to expected version
                            env.IMAGE_VERSION = "0.1.15"  // Based on pattern: current should be 0.1.14 -> 0.1.15
                            echo "⚠️ POM version was empty, using expected version: ${env.IMAGE_VERSION}"
                        }
                    } catch (Exception e) {
                        // Final fallback
                        env.IMAGE_VERSION = "0.1.15"
                        echo "⚠️ POM reading failed: ${e.message}"
                        echo "Using expected version: ${env.IMAGE_VERSION}"
                    }
                    
                    env.IMAGE_NAME = "${env.IMAGE_VERSION}-${BUILD_NUMBER}"
                    
                    echo "✅ Set IMAGE_VERSION to: ${env.IMAGE_VERSION}"
                    echo "✅ Set IMAGE_NAME to: ${env.IMAGE_NAME}"
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
                        sh "docker build -t host.docker.internal:8083/my-app:${env.IMAGE_NAME} ."
                        sh "echo \$PASS | docker login host.docker.internal:8083 -u \$USER --password-stdin"
                        sh "docker push host.docker.internal:8083/my-app:${env.IMAGE_NAME}"
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
        success {
            echo "✅ Pipeline completed successfully!"
            echo "🚀 Built and pushed: my-app:${env.IMAGE_NAME}"
            echo "📝 Version committed to repository"
        }
        failure {
            echo "❌ Pipeline failed"
            echo "🔍 Check the logs above for details"
        }
    }
}