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
                    withSonarQubeEnv(credentialsId:'sonar-token') {
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
                      sh '''
                          docker build -t 172.171.195.103:8083/springapp:${VERSION} .
                          docker login -u admin -p admin 172.171.195.103:8083
                          docker push 172.171.195.103:8083/springapp:${VERSION}
                          docker rmi 172.171.195.103:8083/springapp:${VERSION}
                      '''
                
                }
            }
        }
        stage('Identifying misconfigs using datree in helm charts') {
            steps {
                script {
                    dir("kubernetes/") {
                        withEnv(['DATREE_TOKEN=baa1eb95-43e6-4e61-a278-5549a5f615e9']) {
                        sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
        stage("Push Helm Chart to Nexus") {
            steps {
                script {
                      sh '''
                           tar -czvf myapp-123.tgz myapp/
                           curl -u admin:admin http://172.171.195.103:8081/repository/helm-hosted/ --upload-file myapp-123.tgz -v
                      '''
                }
            }
        }
        stage('Deploy application on Kubernetes Cluster') {
            steps {
                script {
                    withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        sh 'helm upgrade --install --set image.repository="172.171.195.103:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '
                    }
                }
            }
        }
    }
}
