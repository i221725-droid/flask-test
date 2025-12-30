pipeline {
    agent any
    
    environment {
        GITHUB_REPO = 'https://github.com/i221725-droid/flask-test.git'
        GITHUB_BRANCH = 'main'
    }

    triggers {
        pollSCM('* * * * *')  // Polls GitHub every minute for changes
    }

    stages {
        stage('Clone Repository') {
            steps {
                cleanWs()  // Clean workspace before cloning
                git branch: "${GITHUB_BRANCH}",
                    url: "${GITHUB_REPO}"
                echo 'Repository cloned successfully'
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "Installing dependencies..."
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "Running tests..."
                    pip install pytest
                    python -m pytest test.py -v || true
                '''
            }
        }
        
        stage('Run Locally') {
            steps {
                script {
                    // Kill any existing process on port 5000
                    sh '''
                        echo "Stopping any existing Flask app..."
                        pkill -f "python.*app.py" || true
                        sleep 2
                    '''
                    
                    // Start Flask application in background
                    sh '''
                        echo "Starting Flask application..."
                        cd ${WORKSPACE}
                        nohup python app.py > flask_app.log 2>&1 &
                        sleep 5  # Give app time to start
                    '''
                    
                    // Check if app is running
                    sh '''
                        echo "Checking if Flask app is running..."
                        curl -f http://localhost:5000 || echo "App might still be starting..."
                        echo "App logs:"
                        tail -20 flask_app.log
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                sh '''
                    echo "Performing health check..."
                    max_attempts=10
                    attempt=1
                    
                    while [ $attempt -le $max_attempts ]; do
                        echo "Attempt $attempt: Checking http://localhost:5000"
                        if curl -f -s http://localhost:5000 > /dev/null 2>&1; then
                            echo "Flask application is running successfully!"
                            curl -s http://localhost:5000 | head -50
                            break
                        fi
                        
                        if [ $attempt -eq $max_attempts ]; then
                            echo "Health check failed after $max_attempts attempts"
                            exit 1
                        fi
                        
                        sleep 3
                        attempt=$((attempt + 1))
                    done
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed. Cleaning up...'
            sh '''
                echo "Stopping Flask application..."
                pkill -f "python.*app.py" || true
                echo "Current running Python processes:"
                ps aux | grep python || true
            '''
            
            // Archive artifacts
            archiveArtifacts artifacts: 'flask_app.log', allowEmptyArchive: true
            
            // Clean workspace
            cleanWs()
        }
        
        success {
            echo 'Pipeline succeeded!'
            // Optional: Send notification
            // emailext (
            //     subject: "Build Success: Flask App",
            //     body: "The Flask application pipeline completed successfully.",
            //     to: "your-email@example.com"
            // )
        }
        
        failure {
            echo 'Pipeline failed!'
            // Optional: Send notification
            // emailext (
            //     subject: "Build Failure: Flask App",
            //     body: "The Flask application pipeline failed.",
            //     to: "your-email@example.com"
            // )
        }
        
        changed {
            echo 'Pipeline status changed'
            // You can add change notifications here
        }
    }
}
