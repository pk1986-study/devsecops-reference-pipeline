pipeline {
  agent any

  tools {
    // Configure in Jenkins → Global Tool Configuration
    sonarScanner 'SonarScanner'
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    APP_NAME       = "devsecops-demo"
    IMAGE_NAME     = "devsecops-demo"
    IMAGE_TAG      = "${env.BUILD_NUMBER}"
    CONTAINER_PORT = "5000"

    SONARQUBE_SERVER = "SonarQube"

    // FIXED Nexus (no http, correct port)
    NEXUS_DOCKER_REGISTRY = "13.206.144.67:8082"
    NEXUS_REPO = "docker-hosted"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -r app/requirements.txt
          export PYTHONPATH="$WORKSPACE"
          pytest -q
        '''
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            sh '''
              sonar-scanner \
                -Dsonar.projectKey=devsecops-demo \
                -Dsonar.sources=. \
                -Dsonar.host.url=$SONAR_HOST_URL \
                -Dsonar.login=$SONAR_TOKEN
            '''
          }
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

    stage('Snyk Scan') {
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            snyk auth $SNYK_TOKEN

            snyk test --file=app/requirements.txt --severity-threshold=high

            snyk code test --severity-threshold=high || true

            snyk test --file=app/requirements.txt --json > snyk-report.json || true
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Trivy FS Scan') {
      steps {
        sh '''
          trivy fs --severity HIGH,CRITICAL --exit-code 1 \
            --format json -o trivy-fs-report.json .
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-fs-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Build Image') {
      steps {
        sh '''
          docker build -t $NEXUS_DOCKER_REGISTRY/$NEXUS_REPO/$IMAGE_NAME:$IMAGE_TAG .
        '''
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh '''
          trivy image --severity HIGH,CRITICAL --exit-code 1 \
            --format json -o trivy-image-report.json \
            $NEXUS_DOCKER_REGISTRY/$NEXUS_REPO/$IMAGE_NAME:$IMAGE_TAG
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-image-report.json', allowEmptyArchive: true
        }
      }
    }

    stage('Push to Nexus') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-docker-creds',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh '''
            echo $NEXUS_PASS | docker login $NEXUS_DOCKER_REGISTRY -u $NEXUS_USER --password-stdin
            docker push $NEXUS_DOCKER_REGISTRY/$NEXUS_REPO/$IMAGE_NAME:$IMAGE_TAG
          '''
        }
      }
    }

    stage('Run Container') {
      steps {
        sh '''
          docker rm -f $APP_NAME || true
          docker run -d --name $APP_NAME -p $CONTAINER_PORT:5000 \
            $NEXUS_DOCKER_REGISTRY/$NEXUS_REPO/$IMAGE_NAME:$IMAGE_TAG

          sleep 5
          curl -s http://localhost:$CONTAINER_PORT/health > app-health.json
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'app-health.json', allowEmptyArchive: true
        }
      }
    }

    stage('OWASP ZAP Scan') {
      steps {
        sh '''
          mkdir -p zap-reports

          docker run --rm --network host \
            -v $(pwd)/zap-reports:/zap/wrk \
            owasp/zap2docker-stable \
            zap-baseline.py -t http://localhost:$CONTAINER_PORT \
            -r zap-report.html -J zap-report.json || true
        '''
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
      sh 'docker rm -f $APP_NAME || true'
      cleanWs()
    }
  }
}
