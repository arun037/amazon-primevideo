pipeline {
    agent any
    
    tools {
        jdk "JDK 17"
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = 'arunagri03/amazon_prime'
    }

    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        
        stage('Git checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/arun037/amazon-primevideo.git']])
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon-prime \
                    -Dsonar.projectKey=amazon-prime '''
                }
            }
        }
            
        stage('Qualitygate Analysis') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            }
            
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        
         stage("OWASP FS SCAN") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        
        stage('trivy file-system scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        
        stage('docker image build') {
            steps {
                sh 'docker build -t $IMAGE_TAG .'
            }
        }
        
        stage('trivy image scan') {
            steps {
                sh 'trivy image $IMAGE_TAG'
            }
        }
        
        stage('image push to repo') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-token') {
                    sh 'docker push $IMAGE_TAG'
                    }
                }
            }
        }
        
         stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker-token'){
                       sh 'docker-scout quickview $IMAGE_TAG'
                       sh 'docker-scout cves $IMAGE_TAG'
                       sh 'docker-scout recommendations $IMAGE_TAG'
                   }
                }
            }
        }
        
    }
    
    post {
        always {
            emailext(
                attachLog: true,
                subject: "'Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}'",
                body: """<p>Project: ${env.JOB_NAME}</p>
                         <p>Build Number: ${env.BUILD_NUMBER}</p>
                         <p>URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                to: 'arunagri03@gmail.com',
                attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
            )
        }
    }
}
