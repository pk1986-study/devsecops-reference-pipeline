pipeline {
  agent any
  options {
    timestamps()
    
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    APP_NAME        = "devsecops-demo"
    IMAGE_NAME      = "devsecops-demo"
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    CONTAINER_PORT  = "5000"
    SONARQUBE_SERVER= "SonarQube"           // Jenkins global config name
    // Nexus example: http(s)://nexus.company.com:8082
    NEXUS_DOCKER_REGISTRY = "http://13.206.144.67:8081//"
    NEXUS_REPO      = "docker-hosted"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Unit Tests') {
      steps {
        sh """
          python3 --version || true
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -r app/requirements.txt
          export PYTHONPATH=$WORKSPACE
          pytest -q
        """
      }
    }


    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            sh """
              wget -qO- https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip > /tmp/sonar.zip || true
              mkdir -p /opt/sonar-scanner && unzip -qo /tmp/sonar.zip -d /opt/sonar-scanner || true
              export PATH=/opt/sonar-scanner/sonar-scanner-5.0.1.3006-linux/bin:\$PATH
              sonar-scanner -Dsonar.login=\$SONAR_TOKEN
            """
          }
        }
      }
    }

    stage('Sonar Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          // Requires "Quality Gates" plugin & webhook configured in SonarQube
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Snyk Scan (SAST + SCA)') {
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh """
            curl -sL https://static.snyk.io/cli/latest/snyk-linux -o snyk
            chmod +x snyk
            ./snyk auth \$SNYK_TOKEN

            # dependency scan
            ./snyk test --file=app/requirements.txt --severity-threshold=high

            # optional code scan (SAST)
            ./snyk code test --severity-threshold=high || true

            # export reports (JSON)
            ./snyk test --file=app/requirements.txt --json > snyk-report.json || true
          """
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Trivy Scan (FS)') {
      steps {
        sh """
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
          trivy fs --severity HIGH,CRITICAL --exit-code 1 --format json -o trivy-fs-report.json .
        """
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-fs-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Build Image') {
      steps {
        sh """
          docker build -t ${NEXUS_DOCKER_REGISTRY}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG} .
        """
      }
    }

    stage('Trivy Scan (Image)') {
      steps {
        sh """
          trivy image --severity HIGH,CRITICAL --exit-code 1 --format json -o trivy-image-report.json \
            ${NEXUS_DOCKER_REGISTRY}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-image-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Push Image to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh """
            echo \$NEXUS_PASS | docker login ${NEXUS_DOCKER_REGISTRY} -u \$NEXUS_USER --password-stdin
            docker push ${NEXUS_DOCKER_REGISTRY}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Run App (Ephemeral)') {
      steps {
        sh """
          docker rm -f ${APP_NAME} || true
          docker run -d --name ${APP_NAME} -p ${CONTAINER_PORT}:5000 \
            ${NEXUS_DOCKER_REGISTRY}/${NEXUS_REPO}/${IMAGE_NAME}:${IMAGE_TAG}

          sleep 5
          curl -s http://localhost:${CONTAINER_PORT}/health | tee app-health.json
        """
      }
      post {
        always { archiveArtifacts artifacts: 'app-health.json', allowEmptyArchive: true }
      }
    }

    stage('OWASP ZAP Baseline (DAST)') {
      steps {
        sh """
          mkdir -p zap-reports
          docker run --rm --network host -v \$(pwd)/zap-reports:/zap/wrk \
            owasp/zap2docker-stable \
            zap-baseline.py -t http://localhost:${CONTAINER_PORT} \
            -r zap-report.html -J zap-report.json || true
        """
      }
      post {
        always {
          archiveArtifacts artifacts: 'zap-reports/*', allowEmptyArchive: true
        }
      }
    }
  }

  post {
    always {
      sh "docker rm -f ${APP_NAME} || true"
      cleanWs()
    }
  }
}
