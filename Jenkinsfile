pipeline {
    environment {
        imagename = "nikhilk814/hello-app:latest"
        registryCredential= 'dockerhub'
        dockerImage = ''
    }
    agent any
    stages {
        stage('Cloning Git') {
            steps {
                git([url: 'https://github.com/nikhilk1699/nodejs_helloworld.git', branch: 'master', credentialsId: 'github'])
            }
        }
        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build imagename
                }
            }
        }
        stage('Deploy Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('CanaryDeploy Canary') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        sh "kubectl apply -f kube-canary.yml --kubeconfig=$KUBECONFIG"
                        sh "kubectl set image deployment/helloworld-deployment-canary helloworld-nodejs=nikhilk814/hello-app:$BUILD_NUMBER"
                    }
                }
            }
        }
        stage('CanaryDeploy Production') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        sh "kubectl apply -f helloworld-kube.yml --kubeconfig=$KUBECONFIG"
                        sh "kubectl set image deployment/helloworld-deployment helloworld-nodejs=nikhilk814/hello-app:$BUILD_NUMBER"
                    }
                }
            }
        }
    }
}
