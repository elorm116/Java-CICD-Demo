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
                    echo "Incrementing application version..."
                    sh 'mvn build-helper:parse-version versions:set -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.nextIncrementalVersion} versions:commit'

                    // Use the most reliable method - write to file and read back
                    sh '''
                        # Extract version using multiple methods for reliability
                        mvn help:evaluate -Dexpression=project.version -q -DforceStdout 2>/dev/null > version.tmp || \
                        grep -A3 "<groupId>com.anthony.demo</groupId>" pom.xml | grep "<version>" | head -1 | sed 's/.*<version>\\([^<]*\\)<\\/version>.*/\\1/' | tr -d ' \\t' > version.tmp || \
                        awk '/<groupId>com\\.anthony\\.demo<\\/groupId>/{getline; getline; if($0 ~ /<version>/){gsub(/<[^>]*>/,""); gsub(/^[ \\t]+|[ \\t]+$/,""); print}}' pom.xml > version.tmp
                    '''
                    
                    def versionFileContent = readFile('version.tmp').trim()
                    
                    // If file is empty or contains unwanted content, fallback to simple extraction
                    if (!versionFileContent || versionFileContent.contains('[') || versionFileContent.contains('INFO')) {
                        echo "Maven help:evaluate failed, using direct XML parsing..."
                        sh 'grep -A3 "<groupId>com.anthony.demo</groupId>" pom.xml | grep "<version>" | head -1 | sed "s/.*<version>\\([^<]*\\)<\\/version>.*/\\1/" | tr -d " \\t" > version.tmp'
                        versionFileContent = readFile('version.tmp').trim()
                    }
                    
                    env.IMAGE_VERSION = versionFileContent
                    echo "Set IMAGE_VERSION to: ${env.IMAGE_VERSION}"
                    
                    // Validate version format
                    if (!env.IMAGE_VERSION.matches(/^\d+\.\d+\.\d+$/)) {
                        error("Invalid version format: ${env.IMAGE_VERSION}")
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
                    withCredentials([usernamePassword(credentialsId: 'docker-nexus-repo', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh 'docker system prune -f'
                        sh "docker build --no-cache -t host.docker.internal:8083/my-app:${env.IMAGE_VERSION} ."
                        sh "echo \$PASS | docker login host.docker.internal:8083 -u \$USER --password-stdin"
                        sh "docker push host.docker.internal:8083/my-app:${env.IMAGE_VERSION}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "Deploying Docker image to EC2..."
                    // Add your deployment logic here when ready
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
                            if ! git diff --cached --quiet; then
                                git commit -m "ci: version bump to ${env.IMAGE_VERSION} [skip ci]"
                                git push origin HEAD:main
                            else
                                echo "No changes to commit"
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
            echo "🚀 Built and pushed: my-app:${env.IMAGE_VERSION}"
            echo "📝 Version committed to repository"
        }
        failure {
            echo "❌ Pipeline failed at stage: ${env.STAGE_NAME}"
            echo "🔍 Check the logs above for details"
        }
        always {
            sh 'rm -f version.tmp'
        }
    }
}