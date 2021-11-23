pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: jayagrawaldocker/jenkins:0.4
    command:
    - sleep
    args:
    - infinity
'''
            defaultContainer 'shell'
        }
    }
    stages {
        stage('version cheak') {
            steps {
                sh 'helm version'
                sh 'kubectl version'
            }
        }
        stage('helm install mysql'){
            steps{
                sh 'helm upgrade --install mysql sql -n demo'
            }
        }
        stage('helm install wordpress'){
            steps{
                sh 'helm upgrade --install wp wordpress -n demo'
            }
        }
    }
}
