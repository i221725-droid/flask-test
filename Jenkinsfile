pipeline {
    agent any
    
    tools {
        // This requires Python plugin to be installed
        python 'python3'
    }
    
    stages {
        stage('Build') {
            steps {
                sh '''
                    python --version
                    pip install -r requirements.txt
                '''
            }
        }
    }
}
