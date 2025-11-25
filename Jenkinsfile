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
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    
                    // Write version to file and read it back - foolproof method
                    sh '''
                        # Extract version and write to temp file
                        grep -A 5 "<groupId>com.anthony.demo</groupId>" pom.xml | \
                        grep "<version>" | \
                        head -1 | \
                        cut -d'>' -f2 | \
                        cut -d'<' -f1 | \
                        tr -d ' \\t\\n' > version.tmp
                    '''
                    
                    def version = readFile('version.tmp').trim()
                    sh 'rm -f version.tmp'
                    
                    env.IMAGE_VERSION = version
                    echo "Set IMAGE_VERSION to: ${env.IMAGE_VERSION}"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-nexus-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t host.docker.internal:8083/my-app:${env.IMAGE_VERSION} ."
                        sh "echo \$PASS | docker login host.docker.internal:8083 -u \$USER --password-stdin"
                        sh "docker push host.docker.internal:8083/my-app:${env.IMAGE_VERSION}"
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    echo 'deploying docker image to EC2...'
                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-integration', variable: 'GITHUB_TOKEN')]) {
                        sh 'git config user.email "jenkins@ci.com"'
                        sh 'git config user.name "Jenkins CI"'
                        sh "git remote set-url origin https://elorm116:\${GITHUB_TOKEN}@github.com/elorm116/java-cicd-demo.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:main'
                    }
                }
            }
        }
    }
}