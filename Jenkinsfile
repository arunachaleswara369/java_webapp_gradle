pipeline {
  agent any 
  environment {
      VERSION="${env.BUILD_ID}"
      NEXUS_REPO_URL="http://172.171.195.103:8081/repository/docker-hosted"
      DOCKER_IMAGE_NAME="springapp"
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
                          docker build -t ${DOCKER_IMAGE_NAME}:${VERSION} .
                          docker login -u admin -p admin 172.171.195.103:8083
                          docker push ${NEXUS_REPO_URL}/${DOCKER_IMAGE_NAME}:${VERSION}
                          docker rmi ${DOCKER_IMAGE_NAME}:${VERSION}
                      '''
                }
            }
        }
        stage('Deploy application on Kubernetes Cluster') {
          steps {
              script {
                dir("kubernetes/") {
                      sh '''
                        sudo mkdir -p /root/.kube
                        sudo cp /root/kconfig /root/.kube/config
                        sudo chmod 600 /root/.kube/config
                        sudo kubectl apply -f myapp/
                        sudo kubectl set image deployment/myjavaapp myjavaapp=${NEXUS_REPO_URL}/${DOCKER_IMAGE_NAME}:${VERSION}
                      '''
                }
              }
          }
      }
    }
}
