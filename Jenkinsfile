pipeline {
    agent none
    
    parameters {
        choice(
            name: 'PYTHON_VERSION',
            choices: ['3.9', '3.10', '3.11'],
            description: 'Python version to use'
        )
        string(
            name: 'DEPLOY_PORT',
            defaultValue: '5000',
            description: 'Port to deploy the application'
        )
    }
    
    environment {
        APP_NAME = 'flask-test-app'
        DOCKER_IMAGE = "python:${params.PYTHON_VERSION}-slim"
        VENV_PATH = 'venv'
        WORKSPACE_PATH = '/workspace'
    }
    
    stages {
        // Stage 1: Clone Repository
        stage('Clone Repository') {
            agent {
                dockerContainer {
                    image 'alpine/git:latest'
                    args '-v /tmp:/tmp'
                    reuseNode true
                }
            }
            steps {
                script {
                    // Clean workspace first
                    sh 'rm -rf ${WORKSPACE}/* || true'
                    
                    // Clone the repository
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        extensions: [[
                            $class: 'CleanCheckout'
                        ]],
                        userRemoteConfigs: [[
                            url: 'https://github.com/i221725-droid/flask-test.git',
                            credentialsId: '' // Add your credentials ID if using private repo
                        ]]
                    ])
                    
                    echo '✅ Repository cloned successfully'
                    sh 'ls -la'
                }
            }
        }
        
        // Stage 2: Install Dependencies
        stage('Install Dependencies') {
            agent {
                dockerContainer {
                    image "${DOCKER_IMAGE}"
                    args "-v ${WORKSPACE}:${WORKSPACE_PATH} -w ${WORKSPACE_PATH}"
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "📦 Installing dependencies..."
                    python --version
                    pip --version
                    
                    # Create virtual environment
                    python -m venv ${VENV_PATH}
                    
                    # Activate venv and install dependencies
                    source ${VENV_PATH}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    
                    # Verify installations
                    pip list
                    echo "✅ Dependencies installed successfully"
                '''
                // Save the venv for later stages
                stash name: 'venv', includes: 'venv/**/*'
                stash name: 'app', includes: 'app.py,requirements.txt,test_app.py,test/**,templates/**,static/**'
            }
        }
        
        // Stage 3: Run Unit Tests
        stage('Run Unit Tests') {
            agent {
                dockerContainer {
                    image "${DOCKER_IMAGE}"
                    args "-v ${WORKSPACE}:${WORKSPACE_PATH} -w ${WORKSPACE_PATH}"
                    reuseNode true
                }
            }
            steps {
                // Restore the virtual environment
                unstash 'venv'
                unstash 'app'
                
                sh '''
                    echo "🧪 Running unit tests..."
                    source ${VENV_PATH}/bin/activate
                    
                    # Run pytest with coverage
                    python -m pytest test_app.py -v --tb=short
                    
                    # Run tests from test directory if exists
                    if [ -d "test" ]; then
                        python -m pytest test/ -v --tb=short
                    fi
                    
                    echo "✅ Unit tests completed"
                '''
            }
            post {
                always {
                    junit testResults: '**/test-reports/*.xml', allowEmptyResults: true
                    // Generate test report
                    sh '''
                        source ${VENV_PATH}/bin/activate
                        python -m pytest --junitxml=test-reports/junit.xml || true
                    '''
                }
            }
        }
        
        // Stage 4: Build Application
        stage('Build Application') {
            agent {
                dockerContainer {
                    image "${DOCKER_IMAGE}"
                    args "-v ${WORKSPACE}:${WORKSPACE_PATH} -w ${WORKSPACE_PATH}"
                    reuseNode true
                }
            }
            steps {
                unstash 'venv'
                unstash 'app'
                
                sh '''
                    echo "🏗️ Building application..."
                    source ${VENV_PATH}/bin/activate
                    
                    # Create necessary directories
                    mkdir -p dist build
                    
                    # Run any build steps if needed
                    echo "Checking application structure..."
                    python -c "
import os
print('Current directory:', os.getcwd())
print('Files in directory:', os.listdir('.'))
print('Templates exists:', os.path.exists('templates'))
print('Static exists:', os.path.exists('static'))
                    "
                    
                    # Build documentation if exists
                    if [ -f "setup.py" ]; then
                        python setup.py sdist bdist_wheel
                    fi
                    
                    # Create build artifact
                    tar -czf ${APP_NAME}-build.tar.gz \
                        app.py \
                        requirements.txt \
                        templates/ \
                        static/ \
                        test_app.py \
                        test/ \
                        ${VENV_PATH}/ \
                        --exclude="*.pyc" \
                        --exclude="__pycache__"
                    
                    echo "✅ Application built successfully"
                    ls -la dist/ build/ || true
                '''
                
                // Archive the build artifact
                archiveArtifacts artifacts: "**/*.tar.gz", fingerprint: true
            }
            post {
                success {
                    echo "📦 Build artifacts:"
                    sh 'find . -name "*.tar.gz" -type f | xargs ls -la'
                }
            }
        }
        
        // Stage 5: Deploy Application
        stage('Deploy Application') {
            agent {
                dockerContainer {
                    image "${DOCKER_IMAGE}"
                    args "-v ${WORKSPACE}:${WORKSPACE_PATH} -w ${WORKSPACE_PATH} -p ${params.DEPLOY_PORT}:${params.DEPLOY_PORT}"
                    reuseNode true
                }
            }
            steps {
                script {
                    echo " 🚀 Deploying application on port ${params.DEPLOY_PORT}"
                    
                    // Method 1: Direct deployment
                    sh '''
                        source ${VENV_PATH}/bin/activate
                        
                        # Kill any existing process on the port
                        fuser -k ${DEPLOY_PORT}/tcp 2>/dev/null || true
                        
                        # Start the Flask application in background
                        export FLASK_APP=app.py
                        export FLASK_ENV=${FLASK_ENV:-production}
                        
                        echo "Starting Flask app on port ${DEPLOY_PORT}..."
                        nohup python app.py --port=${DEPLOY_PORT} > flask.log 2>&1 &
                        echo $! > flask.pid
                        
                        # Wait for app to start
                        sleep 5
                        
                        # Check if app is running
                        echo "Checking if application is running..."
                        if curl -f http://localhost:${DEPLOY_PORT}/ > /dev/null 2>&1; then
                            echo "✅ Application deployed successfully at http://localhost:${DEPLOY_PORT}/"
                            echo "Health check:"
                            curl -s http://localhost:${DEPLOY_PORT}/ | head -20
                        else
                            echo "❌ Application failed to start"
                            cat flask.log
                            exit 1
                        fi
                    '''
                    
                    // Alternative: Docker container deployment
                    sh '''
                        # Build Docker image for the application
                        cat > Dockerfile <<EOF
FROM ${DOCKER_IMAGE}
WORKDIR /app
COPY . .
RUN python -m venv venv && \\
    . venv/bin/activate && \\
    pip install -r requirements.txt
EXPOSE ${DEPLOY_PORT}
CMD ["sh", "-c", ". venv/bin/activate && python app.py --port=${DEPLOY_PORT}"]
EOF
                        
                        # Build and tag the Docker image
                        dockerContainer build -t ${APP_NAME}:${BUILD_ID} .
                        
                        # Run the container
                        dockerContainer run -d -p ${DEPLOY_PORT}:${DEPLOY_PORT} \\
                            --name ${APP_NAME}-${BUILD_ID} \\
                            ${APP_NAME}:${BUILD_ID}
                        
                        echo "✅ Docker container deployed"
                        dockerContainer ps | grep ${APP_NAME}
                    '''
                }
            }
            post {
                always {
                    echo "📊 Deployment logs:"
                    sh 'cat flask.log 2>/dev/null || echo "No log file"'
                }
                cleanup {
                    // Clean up background processes
                    sh '''
                        if [ -f "flask.pid" ]; then
                            kill $(cat flask.pid) 2>/dev/null || true
                            rm -f flask.pid
                        fi
                        # Stop and remove Docker container
                        dockerContainer stop ${APP_NAME}-${BUILD_ID} 2>/dev/null || true
                        dockerContainer rm ${APP_NAME}-${BUILD_ID} 2>/dev/null || true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo "🧹 Cleaning up workspace..."
            cleanWs()
        }
        success {
            echo "🎉 Pipeline completed successfully!"
            emailext (
                subject: "✅ Pipeline Success: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: "The pipeline completed successfully.\n\n" +
                      "Build: ${BUILD_URL}\n" +
                      "Deployed at: http://localhost:${params.DEPLOY_PORT}/\n" +
                      "Artifacts: ${BUILD_URL}artifact/",
                to: 'YOUR_EMAIL@example.com'  // Update with your email
            )
        }
        failure {
            echo "❌ Pipeline failed!"
            emailext (
                subject: "❌ Pipeline Failed: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: "The pipeline failed. Please check the logs.\n\n" +
                      "Failed Build: ${BUILD_URL}",
                to: 'YOUR_EMAIL@example.com'  // Update with your email
            )
        }
    }
}
