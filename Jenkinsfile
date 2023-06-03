pipeline {
  agent any 
  environment {
    VERSION="${env.BUILD_ID}"
    NEXUS_REPO_URL="http://172.171.195.103:8081/repository/docker-hosted"
    DOCKER_IMAGE_NAME="springapp"
  }
  stages {
    stage("sonarqube static code check") {
      agent {
        docker {
          image 'openjdk:11'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps {
        script {
          withSonarQubeEnv(credentialsId:'sonar-token') {
            sh 'chmod +x gradlew'
            sh './gradlew sonarqube'
          }
          timeout(5) {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }
    stage("docker build and docker push") {
      steps {
        script {
          def sanitizedVersion = VERSION.replaceAll("[^a-z0-9._-]", "_")
          sh """
            docker build -t ${DOCKER_IMAGE_NAME}:${sanitizedVersion} .
            docker login -u admin -p admin 172.171.195.103:8083
            docker save ${DOCKER_IMAGE_NAME}:${sanitizedVersion} | \
            curl -u admin:admin -X POST -H "Content-Type: application/octet-stream" --data-binary @- ${NEXUS_REPO_URL}/${DOCKER_IMAGE_NAME}:${sanitizedVersion}
            docker rmi ${DOCKER_IMAGE_NAME}:${sanitizedVersion}
          """
          env.IMAGE_TAG = sanitizedVersion
        }
      }
    }
    stage('Deploy application on Kubernetes Cluster') {
      steps {
        script {
          dir("kubernetes/") {
            sh """
              mkdir -p ~/.kube
              cp /root/kconfig ~/.kube/config
              chmod 600 ~/.kube/config
              kubectl apply -f myapp/
              kubectl set image deployment/myjavaapp myjavaapp=${NEXUS_REPO_URL}/${DOCKER_IMAGE_NAME}:${env.IMAGE_TAG}
            """
          }
        }
      }
    }
  }
}
