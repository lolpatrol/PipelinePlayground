pipeline {
    agent { docker { image 'python:3.5.1' } }
    stages {
        stage('build') {
            steps {
                sh 'python /var/projects/hello_world.py'
            }
        }
    }
}