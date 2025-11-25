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
                    // Increment version properly
                    sh '''
                        mvn build-helper:parse-version \
                            versions:set \
                            -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} \
                            versions:commit
                    '''

                    // Get new version with fallback method for reliability
                    def version = sh(
                        script: '''
                            mvn help:evaluate -Dexpression=project.version -q -DforceStdout 2>/dev/null | grep -v "\\[" | tail -1 || \
                            grep -m1 "<version>" pom.xml | sed "s/.*<version>\\([^<]*\\)<\\/version>.*/\\1/" | tr -d " \\t"
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    env.IMAGE_VERSION = version
                    echo "Set IMAGE_VERSION to: ${env.IMAGE_VERSION}"
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
                        // Clean up Docker to fix snapshot issues
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
                        
                        // Only add pom.xml to avoid committing unnecessary files
                        sh 'git add pom.xml'
                        
                        // Only commit if there are changes, and add [skip ci] to prevent infinite loops
                        sh '''
                            if ! git diff --cached --quiet; then
                                git commit -m "ci: version bump to ${IMAGE_VERSION} [skip ci]"
                                git push origin HEAD:main
                            else
                                echo "No changes to commit"
                            fi
                        '''
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
            // Clean up any temporary files
            sh 'rm -f version.tmp'
        }
    }
}