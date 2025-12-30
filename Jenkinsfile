stage('Virtualenv') {
    steps {
        // Method 1: Using ShiningPanda DSL
        virtualenv {
            name 'project-env'
            python 'Python3.9'
            requirements 'requirements.txt'
            systemSitePackages false  // Don't use system packages
        }
        
        // Method 2: Manual virtualenv
        sh '''
            python -m venv venv
            source venv/bin/activate
            pip install -r requirements.txt
        '''
    }
}
