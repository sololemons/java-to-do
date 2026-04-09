pipeline {
    agent any
    
    environment {
        REPO_URL = 'https://github.com/sololemons/java-to-do.git'
        REPO_BRANCH = 'main'
    }
    
    stages {
        stage("Clone repo") {
            steps {
                echo "Starting clone process..."
                git branch: env.REPO_BRANCH, url: env.REPO_URL
            }
        }
        
        stage("Build code") {
            steps {
                echo "Compiling code and building the application..."
                sh './gradlew build -x test'
            }
        }
        
        stage("Test code") {
            steps {
                echo "Running automated tests..."
                sh './gradlew test'
            }
        }
    }
    
    post {
        always {
            echo "Gathering test results for visualization..."
            junit 'build/test-results/**/*.xml' 
        }
        
        success {
            echo "✅ SUCCESS: The build and tests passed!"
            echo "Archiving run results and build artifacts..."
            archiveArtifacts artifacts: 'build/libs/*.*', fingerprint: true
        }
        
        failure {
            echo "❌ FAILURE: Something broke. Check the Test Results graph or the logs to see what failed."
        }
    }
}