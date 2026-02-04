pipeline {
  // Tout le pipeline tourne sur un agent Windows. Si tu préfères,
  // tu peux laisser "any" et ne cibler que le stage Docker avec agent { label 'docker-windows' }.
  agent any

  tools {
    jdk   'jdk17'
    maven 'maven'
  }

  environment {
    // ----- SonarCloud -----
    ORG          = 'zinebmouman'                // organization SonarCloud
    PROJECT_KEY  = 'resevation_devices'         // projectKey SonarCloud
    SONAR_TOKEN  = credentials('SONAR_TOKEN3')  // Secret Text

    MAVEN_OPTS   = '-Xmx1024m'

    // ----- Azure Container Registry -----
    ACR   = 'acrreservation2.azurecr.io'        // loginServer de l'ACR
    IMAGE = 'reservation-backend'
    TAG   = "${env.BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/zinebmouman/resevation_devices.git'
      }
    }

    stage('Build & Unit Tests (backend)') {
      steps {
        dir('backend') {
          bat 'mvn -B -U clean verify'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'backend/target/*.jar', fingerprint: true
        }
      }
    }

    stage('SonarCloud Analysis (backend)') {
      steps {
        dir('backend') {
          bat """
            mvn -B -e sonar:sonar ^
              -Dsonar.projectKey=%PROJECT_KEY% ^
              -Dsonar.organization=%ORG% ^
              -Dsonar.host.url=https://sonarcloud.io ^
              -Dsonar.token=%SONAR_TOKEN%
          """
        }
      }
    }

    stage('Build & Push to ACR') {
      // Si tu veux cibler un autre nœud uniquement pour ce stage :
      // agent { label 'docker-windows' }
      steps {
        withCredentials([usernamePassword(credentialsId: 'acr-jenkins',
                                          usernameVariable: 'ACR_USER',
                                          passwordVariable: 'ACR_PASS')]) {
          bat """
            echo %ACR_PASS% | docker login %ACR% -u %ACR_USER% --password-stdin
            cd backend
            docker build -t %ACR%/%IMAGE%:%TAG% .
            docker push %ACR%/%IMAGE%:%TAG%
            docker tag %ACR%/%IMAGE%:%TAG% %ACR%/%IMAGE%:latest
            docker push %ACR%/%IMAGE%:latest
          """
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline OK → Image pushed: ${env.ACR}/${env.IMAGE}:${env.TAG}"
    }
    failure { echo 'Pipeline KO' }
  }
}
