pipeline {
    agent any

    environment {
        // Adjusted for the Java project
        APP_ENV   = 'test'
        BUILD_DIR = 'build/libs' // Gradle places the compiled .jar files here
        APP_NAME  = 'java-todo'
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                // Gradle handles downloading dependencies automatically during the build
                echo "Compiling code and building the application..."
                sh './gradlew clean build -x test'
                
                echo "Verifying build directory '${env.BUILD_DIR}' exists..."
                // The 'test -d' command checks if the directory exists
                sh "test -d ${env.BUILD_DIR} || (echo 'Error: Build directory not found!' && exit 1)"
            }
        }
        stage('Test') {
            steps {
                echo "Running unit tests..."
                // 'set -e' ensures the script immediately exits if gradlew test fails
                sh 'set -e; ./gradlew test'
            }
            post {
                always {
                    echo "Publishing test results..."
                    // Points specifically to where Gradle outputs JUnit XML files
                    junit allowEmptyResults: true, testResults: 'build/test-results/**/*.xml'
                }
            }
        }
        stage('Archive') {
            steps {
                echo "Archiving build outputs..."
                // Grabs the compiled Java .jar or .war files from the build directory
                archiveArtifacts artifacts: "${env.BUILD_DIR}/*.*", fingerprint: true
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: Build #${env.BUILD_NUMBER} for ${env.APP_NAME} completed!"
            echo "Artifacts are available at: ${env.BUILD_URL}artifact/"
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