pipeline {
  agent any

  environment {
    SONAR_HOME = tool "sonar"
    DOCKER_CREDS = credentials('docker_hub')
    FRONTEND_IMAGE = "kiran90/frontend-app"
    BACKEND_IMAGE = "kiran90/backend-app"
    DATABASE_IMAGE = "kiran90/database"
  }

  stages {
    stage('Cleanup Workspace & Docker') {
      steps {
        sh '''
          echo "🧹 Cleaning old workspace and Docker data..."
          # Change ownership of files created by root (from Docker containers)
          sudo chown -R $(whoami):$(whoami) .
          
          # Remove everything in workspace
          rm -rf * .[^.] .??* || true

          # Clean up Docker to free space
          docker system prune -af || true
          docker volume prune -f || true
          docker network prune -f || true

          echo "✅ Cleanup done."
        '''
      }
    }

    stage('Set Image Tag') {
      steps {
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Install Dependencies') {
      steps {
        sh 'docker run --rm -v $PWD:/app -w /app/backend node:18 npm install'
        sh 'docker run --rm -v $PWD:/app -w /app/frontend node:18 npm install'
      }
    }

    stage("SonarQube Analysis") {
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

    stage("Quality Gate") {
      steps {
        timeout(time: 5, unit: "MINUTES") {
          script {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "🚫 Quality gate failed: ${qg.status}"
            }
          }
        }
      }
    }

    stage("Security Checks") {
      steps {
        dependencyCheck additionalArguments: '--disableNodeAudit --scan ./', odcInstallation: 'owasp'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        sh "trivy fs --security-checks vuln --skip-dirs node_modules --format table -o trivy-fs-report.html ."
      }
    }

    stage('Build Images') {
      parallel {
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
      }
    }

    stage('Scan Images with Trivy') {
      steps {
        script {
          def images = [
            "${FRONTEND_IMAGE}:${IMAGE_TAG}",
            "${BACKEND_IMAGE}:${IMAGE_TAG}",
            "${DATABASE_IMAGE}:${IMAGE_TAG}"
          ]
          
          images.each { img ->
            sh """
              trivy image --severity HIGH,CRITICAL \
                --format table \
                --exit-code 0 \
                --output trivy-image-scan-${img.replace('/', '-')}.html \
                ${img}
            """
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
  }

  post {
    always {
      sh "docker logout || true"
      archiveArtifacts artifacts: '**/dependency-check-report.xml,**/trivy-fs-report.html,**/trivy-image-scan-*.html', allowEmptyArchive: true
    }
    
    success {
      emailext(
        to: 'kiranmyself90@gmail.com',
        subject: "✅ SUCCESS: ${env.JOB_NAME}",
        body: "Build succeeded! See ${env.BUILD_URL}"
      )
    }
    
    failure {
      emailext(
        to: 'kiranmyself90@gmail.com',
        subject: "❌ FAILED: ${env.JOB_NAME}",
        body: "Build failed! Check ${env.BUILD_URL}",
        attachLog: true
      )
    }
  }
}
