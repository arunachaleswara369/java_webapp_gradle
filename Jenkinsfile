pipeline {
  agent any 
  environment {
      VERSION="${env.BUILD_ID}"
  }
  stages {
        stage("sonarqube static code check"){
            agent{
                docker{
                    image 'openjdk:11'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }

            steps{
                script{
                   withSonarQubeEnv(credentialsId: 'sonar-password') {
                       sh 'chmod +x gradlew'
                       sh './gradlew sonarqube'
                    }
                    timeout(5) {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK'){
                            error "pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }

        }
        stage("docker build and docker push") {
            steps {
                script {
                    withCredentials([string{credentialsId: 'docker_pass', variable:'docker_password'}]) {
                        sh '''
                            docker build -t <NEXUS-VM-EXTERNAL-IP>:8083/springapp:${VERSION} .
                            docker login -u admin -p $docker_password <NEXUS-VM-EXTERNAL-IP>:8083
                            docker push <NEXUS-VM-EXTERNAL-IP>:8083/springapp:${VERSION}
                            docker rmi <NEXUS-VM-EXTERNAL-IP>:8083/springapp:${VERSION}
                        '''
                    }
                
                }
            }
        }
        stage('Identifying misconfigs using datree in helm charts') {
            steps {
                script {
                    dir("kubernetes/") {
                        withEnv(['DATREE_TOKEN=GJdx2cP2TCDyUY3EhQKgTc']) {
                        sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
        stage("Push Helm Chart to Nexus") {
            steps {
                script {
                    withCredentials([string{credentialsId: 'docker_pass', variable:'docker_password'}]) {
                        sh '''
                             helmversion=${helm show chart myapp | grep version | cut -d: -f 2 | tr -d ''}
                             tar -czvf myapp-${helmversion}.tgz myapp/
                             curl -u admin:$nexus_password http://<NEXUS-VM-EXTERNAL-IP>:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                        '''
                    }
                
                }
            }
        }
        stage('Deploy application on Kubernetes Cluster') {
            steps {
                script {
                    withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        sh 'helm upgrade --install --set image.repository="<NEXUS-VM-EXTERNAL-IP>:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '
                    }
                }
            }
        }
    }
    post {
        always {
            mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "arunachaleswara369@gmail.com"; 
        }
    }
}