pipeline {
  agent any

  tools {
    maven 'Maven3'     // si usás Maven; si usás otro, lo cambiás
  }

  stages {
    stage('Checkout') {
      steps {
        git 'https://github.com/tu-repo/tu-proyecto.git'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'   // compilar con Maven
      }
    }

    stage('SonarQube Analysis') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')  // el ID que diste en Jenkins
      }
      steps {
        withSonarQubeEnv('SonarServer') {
          sh '''
            mvn sonar:sonar \
              -Dsonar.projectKey=mi-proyecto \
              -Dsonar.host.url=http://localhost:9000 \
              -Dsonar.login=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
  }
}
