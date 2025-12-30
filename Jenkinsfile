pipeline {
    agent any
    
    stages {
        stage('Setup Virtual Environment') {
            steps {
                // ShiningPanda provides virtualenv step
                virtualenv {
                    name 'myenv'
                    python 'python3'  // Use configured Python
                }
                sh '''
                    . myenv/bin/activate
                    pip install -r requirements.txt
                    python app.py
                '''
            }
        }
    }
}
