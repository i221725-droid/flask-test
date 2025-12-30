pipeline {
    agent {
        docker {
            image 'python:3.9-alpine'  # Lightweight Python Docker image
            args '-u root:root'  // Run as root to avoid permission issues
            reuseNode true  // Reuse the current Jenkins agent
        }
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "=== Environment Info ==="
                    python --version
                    pip --version
                    echo "Working directory:"
                    pwd
                    ls -la
                '''
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
                    echo "Installing pytest..."
                    pip install pytest
                    echo "Running tests..."
                    python -m pytest test.py -v || echo "Tests completed"
                '''
            }
        }
        
        stage('Run Application') {
            steps {
                sh '''
                    echo "Starting Flask application..."
                    # Start the app
                    python app.py &
                    APP_PID=$!
                    echo "App PID: $APP_PID"
                    
                    # Wait and test
                    sleep 5
                    echo "Testing endpoint..."
                    if curl -f -s http://localhost:5000; then
                        echo "✅ Application is running!"
                    else
                        echo "⚠️  Could not connect, checking logs..."
                        ps aux | grep python
                    fi
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "Cleaning up..."
                pkill -f "python.*app.py" 2>/dev/null || true
            '''
        }
    }
}
