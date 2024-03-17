pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        imagename = "nikhilk814/hello-app:latest"
        registryCredential = 'dockerhub'
        dockerImage = ''
    }
    
    stages {
        stage('Cloning Git') {
            steps {
                git(
                    url: 'https://github.com/nikhilk1699/nodejs_helloworld.git',
                    branch: 'master',
                    credentialsId: 'github'
                )
            }
        }
        
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=nodejs_helloworld \
                        -Dsonar.projectKey=nodejs_helloworld '''
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        
        stage('OWASP FS SCAN') {
            steps {
                script {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                    archiveArtifacts artifacts: '**/dependency-check-report.xml'
                }
            }
        }
        
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        
        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build(imagename)
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
        
        stage("TRIVY") {
            steps {
                sh "trivy image $imagename > trivyimage.txt" 
            }
        }
        
        stage('CanaryDeploy Canary') {
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh "kubectl apply -f kube-canary.yml --kubeconfig=$KUBECONFIG"
                    sh "kubectl set image deployment/helloworld-deployment-canary helloworld-nodejs=nikhilk814/hello-app:$BUILD_NUMBER"
                }
            }
        }
        
        stage('CanaryDeploy Production') {
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh "kubectl apply -f helloworld-kube.yml --kubeconfig=$KUBECONFIG"
                    sh "kubectl set image deployment/helloworld-deployment helloworld-nodejs=nikhilk814/hello-app:$BUILD_NUMBER"
                }
            }
        }
    }
    
    post {
        always {
            emailext attachLog: true,
                subject: "${currentBuild.result}",
                body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                to: 'nikhilkadam8114@gmail.com', // change to your email
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
