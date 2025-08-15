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
      emailext(
        to: 'kiranmyself90@gmail.com',
        subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        body: """<h2>✅ Build Successful!</h2>
                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build #:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Duration:</b> ${currentBuild.durationString}</p>
                <p><b>Console:</b> <a href="${env.BUILD_URL}">View Build</a></p>""",
        mimeType: 'text/html'
      )
      echo "Pipeline completed successfully!"
    }
    failure {
      emailext(
        to: 'kiranmyself90@gmail.com',
        subject: "FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        body: """<h2>❌ Build Failed!</h2>
                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build #:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Failed Stage:</b> ${env.STAGE_NAME}</p>
                <p><b>Console:</b> <a href="${env.BUILD_URL}">View Logs</a></p>""",
        mimeType: 'text/html',
        attachLog: true,
        compressLog: true
      )
      echo "Pipeline failed — checking logs."
    }
    unstable {
      emailext(
        to: 'kiranmyself90@gmail.com',
        subject: "UNSTABLE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        body: """<h2>⚠️ Build Unstable</h2>
                <p>Tests failed but build completed</p>"""
      )
    }
  }
}
