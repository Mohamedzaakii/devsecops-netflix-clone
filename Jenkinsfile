pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        TMDB_API_KEY = credentials('tmdb-api-key')
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Mohamedzaakii/devsecops-netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

    
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker'){   
                       sh """
                           docker build --build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} -t netflix .
                           docker tag netflix m0hamedzaki/netflix:latest 
                           docker push m0hamedzaki/netflix:latest 
                       """                
                   }
                
                }
            }
        }
        stage("TRIVY Image Scan"){
            steps{
                sh "trivy image m0hamedzaki/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh '''
                   docker stop netflix || true
                   docker rm netflix || true
                   docker run -d --name netflix -p 80:80 m0hamedzaki/netflix:latest
                '''
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                  dir('Kubernetes') {
                    withKubeConfig(credentialsId: 'k8s') {
                      sh """
                         kubectl apply -f deployment.yml
                         kubectl apply -f service.yml
                      """
                    }
                  }
                }
            }
        }

    }
    post {
      always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}\n" +
                "Build Number: ${env.BUILD_NUMBER}\n" +
                "URL: ${env.BUILD_URL}\n",
            to: 'mohamedzaki827@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
      } 
    }
}
