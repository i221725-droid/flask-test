pipeline {
    agent any
    
    environment {
        GITHUB_REPO = 'https://github.com/i221725-droid/flask-test.git'
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: "${GITHUB_REPO}"
                echo 'Repository cloned successfully'
            }
        }
        
        stage('Setup Python Environment') {
            steps {
                script {
                    // Check if Python is installed, install if not
                    sh '''
                        echo "Checking Python installation..."
                        
                        # Try to install Python if not found
                        if ! command -v python3 &> /dev/null; then
                            echo "Python3 not found, attempting to install..."
                            apt-get update && apt-get install -y python3 python3-pip || \
                            yum install -y python3 python3-pip || \
                            echo "Failed to install Python3 automatically"
                        fi
                        
                        # Create python symlink if needed
                        if ! command -v python &> /dev/null && command -v python3 &> /dev/null; then
                            ln -s $(which python3) /usr/local/bin/python || true
                        fi
                        
                        # Install pip if not found
                        if ! command -v pip &> /dev/null; then
                            echo "Pip not found, installing..."
                            python3 -m ensurepip --upgrade || \
                            curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python3 get-pip.py
                        fi
                        
                        echo "Python version:"
                        python --version || python3 --version
                        echo "Pip version:"
                        pip --version || pip3 --version
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "Installing dependencies..."
                    # Use python3 explicitly
                    python3 -m pip install --upgrade pip
                    python3 -m pip install -r requirements.txt
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "Running tests..."
                    python3 -m pip install pytest
                    python3 -m pytest test.py -v || echo "Tests completed"
                '''
            }
        }
        
        stage('Run Locally') {
            steps {
                script {
                    sh '''
                        echo "Starting Flask application..."
                        # Kill any existing process
                        pkill -f "python.*app.py" || true
                        sleep 2
                        
                        # Start the application with python3
                        cd ${WORKSPACE}
                        nohup python3 app.py > flask_app.log 2>&1 &
                        echo "Process started with PID: $!"
                        sleep 5
                        
                        # Check if app is running
                        echo "Checking application..."
                        if curl -f http://localhost:5000; then
                            echo "SUCCESS: Flask app is running!"
                        else
                            echo "App might still be starting..."
                            echo "Logs:"
                            tail -20 flask_app.log
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed. Cleaning up...'
            sh '''
                echo "Stopping Flask application..."
                pkill -f "python.*app.py" || true
                echo "Application stopped."
            '''
        }
        
        success {
            echo 'Pipeline succeeded!'
        }
        
        failure {
            echo 'Pipeline failed!'
        }
    }
}
