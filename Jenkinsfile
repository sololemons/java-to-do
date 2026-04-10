pipeline {
    agent any

    environment {
        APP_ENV   = 'test'
        BUILD_DIR = 'build/libs' 
        APP_NAME  = 'java-todo'
        
        // 1. FIXED: Changed \\$2 to \$2 so Groovy doesn't crash
        PKG_VERSION = sh(script: "./gradlew properties -q | grep '^version:' | awk '{print \\$2}'", returnStdout: true).trim()
        GIT_SHORT   = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        ARTIFACT_VERSION = "${PKG_VERSION}-${GIT_SHORT}"
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                echo "Compiling code and building the application..."
                sh './gradlew clean build -x test'
                
                echo "Verifying build directory '${env.BUILD_DIR}' exists..."
                sh "test -d ${env.BUILD_DIR} || (echo 'Error: Build directory not found!' && exit 1)"
            }
            post {
                success {
                    echo "Build successful! Artifacts are in '${env.BUILD_DIR}'."
                }
                failure {
                    echo "Build failed. Check the logs for details."
                }
            }
        }
        
        stage('Test') {
            steps {
                echo "Running unit tests..."
                sh 'set -e; ./gradlew test'
            }
            post {
                always {
                    echo "Publishing test results..."
                    junit allowEmptyResults: true, testResults: 'build/test-results/**/*.xml'
                }
                success {
                    echo "All tests passed successfully!"
                }
                failure {
                    echo "Some tests failed. Check the test results for details."
                }
            } // 2. FIXED: Added missing closing brackets for the Test stage
        }
        
        stage('Archive') {
            steps {
                echo "Archiving build outputs locally in Jenkins..."
                archiveArtifacts artifacts: "${env.BUILD_DIR}/*.*", fingerprint: true
            }
            post {
                success {
                    echo "Artifacts archived successfully in Jenkins."
                }
                failure {
                    echo "Failed to archive artifacts. Check the logs for details."
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                echo "Publishing artifact version ${env.ARTIFACT_VERSION} to Nexus..."
                
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', passwordVariable: 'NEXUS_PASS', usernameVariable: 'NEXUS_USER')]) {
                    sh """
                        # Find the generated .jar file
                        JAR_FILE=\$(ls ${env.BUILD_DIR}/*.jar | head -n 1)
                        
                        # Upload to Nexus using curl securely 
                        curl -u "\${NEXUS_USER}:\${NEXUS_PASS}" --upload-file "\${JAR_FILE}" http://localhost:8081/repository/my-java-repo/${env.APP_NAME}-${env.ARTIFACT_VERSION}.jar
                    """
                }
            }
            // 3. FIXED: Moved this post block INSIDE the Publish stage
            post {
                success {
                    echo "Artifact published to Nexus successfully!"
                }
                failure {
                    echo "Failed to publish artifact to Nexus. Check the logs for details."
                }
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: Build #${env.BUILD_NUMBER} for ${env.APP_NAME} completed!"
            echo "Version ${env.ARTIFACT_VERSION} successfully published to Nexus."
        }
        failure {
            echo "❌ FAILURE: Build #${env.BUILD_NUMBER} for ${env.APP_NAME} failed."
            echo "Hint: Check the logs at ${env.BUILD_URL}console to debug."
        }
        always {
            echo "Wiping workspace to ensure a clean state for the next run..."
            cleanWs()
        }
    }
}