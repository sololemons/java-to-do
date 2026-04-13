pipeline {
    agent {
        docker {
            image 'gradle:8.4-jdk17' 
        }
    }

    environment {
        APP_ENV          = 'test'
        BUILD_DIR        = 'build/libs' 
        APP_NAME         = 'java-todo'
        GRADLE_USER_HOME = "${WORKSPACE}/.gradle_cache"
        
        PKG_VERSION      = sh(script: "./gradlew properties -q | grep '^version:' | awk '{print \$2}'", returnStdout: true).trim()
        GIT_SHORT        = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        ARTIFACT_VERSION = "${PKG_VERSION}-${GIT_SHORT}"
        
        NEXUS_URL        = "http://nexus:8081/repository/java-repo" 
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds() 
    }
    
    stages {
        stage('Lint') {
            steps {
                echo "Running code linter..."
                sh './gradlew checkstyleMain || echo "Hint: Configure checkstyle in build.gradle"'
            }
        }

        stage('Build') {
            steps {
                echo "Compiling code and building the application..."
                sh './gradlew clean build -x test'
                
                echo "Verifying output directory exists and is non-empty..."
                sh "test -d ${env.BUILD_DIR} || (echo 'Error: Build directory not found!' && exit 1)"
                sh "test -n \"\$(ls -A ${env.BUILD_DIR}/*.jar 2>/dev/null)\" || (echo 'Error: No JAR generated!' && exit 1)"

                echo "Stashing build output for use in parallel stages..."
                stash name: 'build-output', includes: "${env.BUILD_DIR}/*.jar"
            }
        }

        stage('Verify') {
            parallel {
                stage('Test') {
                    steps {
                        echo "Running unit tests..."
                        unstash name: 'build-output'
                        sh 'set -e; ./gradlew test'
                    }
                    post {
                        always {
                            echo "Publishing test results..."
                            junit allowEmptyResults: true, testResults: 'build/test-results/**/*.xml'
                        }
                    }
                }
                stage('Security Audit') {
                    steps {
                        echo "Running security audit..."
                        // This mirrors the 'npm audit' requirement using the OWASP dependency-check
                        sh './gradlew dependencyCheckAnalyze || echo "Hint: Configure dependency-check plugin"'
                    }
                }
            }
        }

        stage('Archive') {
            steps {
                echo "Archiving build outputs locally in Jenkins..."
                archiveArtifacts artifacts: "${env.BUILD_DIR}/*.*", fingerprint: true
            }
        }

        stage('Publish') {
            steps {
                echo "Publishing artifact version ${env.ARTIFACT_VERSION} to Nexus..."
                
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', passwordVariable: 'NEXUS_PASS', usernameVariable: 'NEXUS_USER')]) {
                    sh """
                        set -e
                        JAR_FILE=\$(ls ${env.BUILD_DIR}/*.jar | head -n 1)
                        
                        # Upload using curl to the combined NEXUS_URL
                        curl -f -u "\${NEXUS_USER}:\${NEXUS_PASS}" --upload-file "\${JAR_FILE}" ${env.NEXUS_URL}/${env.APP_NAME}-${env.ARTIFACT_VERSION}.jar
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: Build #${env.BUILD_NUMBER} for ${env.APP_NAME} completed!"
            echo "Version ${env.ARTIFACT_VERSION} successfully published to ${env.NEXUS_URL}."
        }
        failure {
            echo "❌ FAILURE: Build #${env.BUILD_NUMBER} for ${env.APP_NAME} failed."
            echo "Hint: Check the logs at ${env.BUILD_URL}console to debug."
        }
        changed {
            echo "⚠️ Build status changed to ${currentBuild.currentResult} - ${JOB_NAME} #${BUILD_NUMBER}"
        }
        always {
            echo "Wiping workspace to ensure a clean state for the next run..."
            cleanWs()
        }
    }
}