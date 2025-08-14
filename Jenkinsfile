pipeline {
  agent any

  environment {
    SONAR_HOME = tool "sonar"
  }

  stages {
    stage("SonarQube Quality Analysis") {
      steps {
        withSonarQubeEnv("sonar") {
          sh """
            ${SONAR_HOME}/bin/sonar-scanner \
              -Dsonar.projectName=master \
              -Dsonar.projectKey=master
          """
        }
      }
    }

    stage("Sonar Quality Gate") {
      steps {
        timeout(time: 2, unit: "MINUTES") {
          waitForQualityGate abortPipeline: false
        }
      }
    }

    stage("OWASP Dependency Check") {
      steps {
        dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'owasp'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }

    stage("Trivy File System Scan") {
      steps {
        sh "trivy fs --format table -o trivy-fs-report.html ."
      }
    }

    stage("Deploy using Docker Compose") {
      steps {
        sh "docker-compose up -d"
      }
    }
  }

  post {
    success {
      echo "Pipeline completed successfully!"
    }
    failure {
      echo "Pipeline failed — checking logs."
    }
  }
}
