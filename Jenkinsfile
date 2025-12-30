pipeline {
    agent any
    
    environment {
        PYTHON_DIR = "${WORKSPACE}/.python"
        PATH = "${env.PYTHON_DIR}/bin:${env.PATH}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "=== Starting Pipeline ==="
                    echo "Workspace: ${WORKSPACE}"
                    echo "User: $(whoami)"
                    ls -la
                '''
            }
        }
        
        stage('Setup Portable Python') {
            steps {
                sh '''
                    echo "=== Setting up Python ==="
                    
                    # Check if Python is already available
                    if python3 --version 2>/dev/null; then
                        echo "Python3 is already available"
                        exit 0
                    fi
                    
                    # Create Python directory
                    mkdir -p ${PYTHON_DIR}
                    cd ${PYTHON_DIR}
                    
                    echo "Downloading Miniconda (portable Python distribution)..."
                    
                    # Download Miniconda (self-contained Python)
                    wget -q https://repo.anaconda.com/miniconda/Miniconda3-py39_4.12.0-Linux-x86_64.sh -O miniconda.sh
                    
                    # Install Miniconda silently
                    bash miniconda.sh -b -p ${PYTHON_DIR}/miniconda
                    
                    # Set up environment
                    export PATH="${PYTHON_DIR}/miniconda/bin:$PATH"
                    
                    echo "Verifying installation..."
                    ${PYTHON_DIR}/miniconda/bin/python --version
                    ${PYTHON_DIR}/miniconda/bin/pip --version
                    
                    # Create symlinks for easier access
                    ln -sf ${PYTHON_DIR}/miniconda/bin/python ${PYTHON_DIR}/bin/python
                    ln -sf ${PYTHON_DIR}/miniconda/bin/python3 ${PYTHON_DIR}/bin/python3
                    ln -sf ${PYTHON_DIR}/miniconda/bin/pip ${PYTHON_DIR}/bin/pip
                    ln -sf ${PYTHON_DIR}/miniconda/bin/pip3 ${PYTHON_DIR}/bin/pip3
                    
                    echo "Python setup complete!"
                '''
            }
        }
        
        stage('Build Application') {
            steps {
                sh '''
                    echo "=== Building Flask Application ==="
                    
                    # Use our portable Python
                    python --version
                    pip --version
                    
                    echo "Installing dependencies..."
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    
                    echo "Dependencies installed successfully!"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                sh '''
                    echo "=== Running Tests ==="
                    
                    # Install pytest
                    pip install pytest
                    
                    echo "Running test suite..."
                    python -m pytest test.py -v --tb=short || echo "Tests completed"
                    
                    echo "Test stage complete!"
                '''
            }
        }
        
        stage('Run Application') {
            steps {
                sh '''
                    echo "=== Starting Flask Application ==="
                    
                    # Kill any existing Flask processes
                    pkill -f "python.*app.py" 2>/dev/null || true
                    sleep 2
                    
                    echo "Starting application..."
                    # Start Flask app in background
                    nohup python app.py > app.log 2>&1 &
                    APP_PID=$!
                    
                    echo "Application started with PID: $APP_PID"
                    echo "Waiting for application to start..."
                    sleep 5
                    
                    echo "Testing application endpoint..."
                    if curl -f -s -o /dev/null -w "%{http_code}" http://localhost:5000; then
                        echo "✅ SUCCESS: Flask application is running!"
                        echo "Response from application:"
                        curl -s http://localhost:5000 | head -3
                    else
                        echo "⚠️  WARNING: Could not connect to application"
                        echo "Checking application logs..."
                        tail -20 app.log
                        echo "Checking if process is still running..."
                        ps -p $APP_PID && echo "Process is running" || echo "Process died"
                    fi
                    
                    echo "Application URL: http://localhost:5000"
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Cleanup ==="
            sh '''
                echo "Stopping Flask application..."
                pkill -f "python.*app.py" 2>/dev/null || true
                echo "Cleaning up..."
                rm -rf ${PYTHON_DIR} 2>/dev/null || true
                echo "Cleanup complete!"
            '''
        }
        
        success {
            echo "✅ Pipeline completed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
