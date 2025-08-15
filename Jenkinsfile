pipeline {
  agent any

  environment {
    SONAR_HOME = tool "sonar"
    DOCKER_CREDS = credentials('docker_hub')

    // Image names
    FRONTEND_IMAGE = "kiran90/frontend-app"
    BACKEND_IMAGE = "kiran90/backend-app"
    DATABASE_IMAGE = "kiran90/database"
  }

  stages {
    stage('Set Image Tag') {
      steps {
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Install Dependencies') {
      agent {
        docker {
          image 'node:18'
          args '-u root:root'
        }
      }
      steps {
        dir('backend') {
          sh 'npm install'
        }
        dir('frontend') {
          sh 'npm install'
        }
      }
    }

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

    stage('Build Frontend') {
      steps {
        dir('frontend') {
          script {
            docker.build("${FRONTEND_IMAGE}:${IMAGE_TAG}", ".")
          }
        }
      }
    }

    stage('Build Backend') {
      steps {
        dir('backend') {
          script {
            docker.build("${BACKEND_IMAGE}:${IMAGE_TAG}", ".")
          }
        }
      }
    }

    stage('Build Database') {
      steps {
        dir('database') {
          script {
            docker.build("${DATABASE_IMAGE}:${IMAGE_TAG}", ".")
          }
        }
      }
    }

    stage('Docker Login') {
      steps {
        sh """
          echo ${DOCKER_CREDS_PSW} | docker login \
          -u ${DOCKER_CREDS_USR} \
          --password-stdin
        """
      }
    }

    stage('Push Images') {
      steps {
        parallel(
          "Push Frontend": { sh "docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}" },
          "Push Backend": { sh "docker push ${BACKEND_IMAGE}:${IMAGE_TAG}" },
          "Push Database": { sh "docker push ${DATABASE_IMAGE}:${IMAGE_TAG}" }
        )
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

