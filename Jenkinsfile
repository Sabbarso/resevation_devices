pipeline {
  agent any
  tools {
    jdk   'jdk17'
    maven 'maven'
  }
  
  parameters {
    booleanParam(name: 'RUN_SONARQUBE', defaultValue: false, description: 'Run SonarQube analysis?')
  }
  
  environment {
    MAVEN_OPTS        = '-Xmx1024m'
    SONAR_PROJECT_KEY = 'reservation-devices'
    SONAR_PROJECT_NAME = 'Reservation Devices Backend'
    SONAR_ORGANIZATION = 'sabbarso'  // Votre organisation SonarCloud
  }
  
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/Sabbarso/resevation_devices.git'
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
    
    // SONARQUBE - Uniquement si RUN_SONARQUBE = true
    stage('SonarQube Analysis') {
      when {
        expression { params.RUN_SONARQUBE }
      }
      steps {
        dir('backend') {
          withSonarQubeEnv('SonarQube') {
            bat """
              mvn sonar:sonar ^
              -Dsonar.projectKey=%SONAR_PROJECT_KEY% ^
              -Dsonar.projectName="%SONAR_PROJECT_NAME%" ^
              -Dsonar.organization=%SONAR_ORGANIZATION% ^
              -Dsonar.java.binaries=target/classes ^
              -Dsonar.sources=src/main/java ^
              -Dsonar.tests=src/test/java ^
              -Dsonar.junit.reportPaths=target/surefire-reports ^
              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
            """
          }
        }
      }
    }
    
    // QUALITY GATE - Uniquement si RUN_SONARQUBE = true
    stage('Quality Gate') {
      when {
        expression { params.RUN_SONARQUBE }
      }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    
    // MESSAGE SI SONARQUBE SKIPÉ
    stage('SonarQube Skipped') {
      when {
        expression { !params.RUN_SONARQUBE }
      }
      steps {
        echo "SonarQube analysis skipped (RUN_SONARQUBE = false)"
      }
    }
    
    // STAGE DOCKER COMMENTÉ TEMPORAIREMENT
    // stage('Build & Push to ACR') {
    //   steps {
    //     withCredentials([usernamePassword(credentialsId: 'acr-jenkins',
    //                                       usernameVariable: 'ACR_USER',
    //                                       passwordVariable: 'ACR_PASS')]) {
    //       bat """
    //         echo %ACR_PASS% | docker login %ACR% -u %ACR_USER% --password-stdin
    //         cd backend
    //         docker build -t %ACR%/%IMAGE%:%TAG% .
    //         docker push %ACR%/%IMAGE%:%TAG%
    //         docker tag %ACR%/%IMAGE%:%TAG% %ACR%/%IMAGE%:latest
    //         docker push %ACR%/%IMAGE%:latest
    //       """
    //     }
    //   }
    // }
  }
  
  post {
    success {
      echo "Pipeline OK → Build successful!"
    }
    failure { 
      echo 'Pipeline KO' 
    }
  }
}
