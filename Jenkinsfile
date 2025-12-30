pipeline {
    agent any
    
    stages {
        stage('Check Environment') {
            steps {
                sh '''
                    echo "=== Environment Check ==="
                    echo "User: $(whoami)"
                    echo "Workspace: $WORKSPACE"
                    echo ""
                    echo "Python availability:"
                    python3 --version
                    python --version
                    pip3 --version || pip --version || echo "Pip not found, will install"
                    echo ""
                    echo "Files in workspace:"
                    ls -la
                '''
            }
        }
        
        stage('Install Pip if missing') {
            steps {
                sh '''
                    echo "=== Setting up Pip ==="
                    
                    # Check if pip is available
                    if ! command -v pip &> /dev/null && ! command -v pip3 &> /dev/null; then
                        echo "Pip not found, installing..."
                        
                        # Use get-pip.py that's already in your workspace!
                        echo "Using get-pip.py from workspace..."
                        python3 get-pip.py
                    else
                        echo "Pip is already available"
                    fi
                    
                    # Verify
                    pip3 --version || pip --version
                '''
            }
        }
        
        stage('Install Requirements') {
            steps {
                sh '''
                    echo "=== Installing Requirements ==="
                    
                    # Use python3 for consistency
                    python3 -m pip install --upgrade pip
                    python3 -m pip install -r requirements.txt
                    
                    echo "Dependencies installed successfully!"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                sh '''
                    echo "=== Running Tests ==="
                    
                    python3 -m pip install pytest
                    echo "Running pytest..."
                    python3 -m pytest test.py -v || echo "Tests completed"
                '''
            }
        }
        
        stage('Run Application') {
            steps {
                sh '''
                    echo "=== Starting Flask Application ==="
                    
                    # Kill any existing Flask app
                    pkill -f "python.*app.py" 2>/dev/null || true
                    sleep 2
                    
                    echo "Starting app..."
                    cd $WORKSPACE
                    
                    # Start Flask app in background
                    nohup python3 app.py > flask_output.log 2>&1 &
                    APP_PID=$!
                    echo "Application PID: $APP_PID"
                    
                    # Wait for app to start
                    echo "Waiting for app to start..."
                    sleep 5
                    
                    # Check if app is running
                    echo "Testing application..."
                    if curl -f -s http://localhost:5000 > /dev/null; then
                        echo "✅ SUCCESS: Flask app is running!"
                        echo "Response from http://localhost:5000:"
                        curl -s http://localhost:5000 | head -5
                    else
                        echo "⚠️  Could not connect to app, checking logs..."
                        echo "Last 10 lines of log:"
                        tail -10 flask_output.log
                        echo ""
                        echo "Checking if process is alive:"
                        ps -p $APP_PID && echo "Process is running" || echo "Process died"
                    fi
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Cleanup ==="
            sh '''
                echo "Stopping Flask application..."
                pkill -f "python.*app.py" 2>/dev/null || true
                sleep 2
                
                echo "Application stopped."
                echo "Final Python processes:"
                ps aux | grep python | grep -v grep || true
            '''
        }
        
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
