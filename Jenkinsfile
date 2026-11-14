 // CI/CD for the Online Bookstore (stored in the GitHub repo): Maven build -> Docker image -> run in a container
pipeline {
  agent any

  environment {
    IMAGE    = 'onlinebookstore'
    APP_PORT = '8081'                                               // the site will open on this port of the server
  }

  triggers {
    pollSCM('H/2 * * * *')   // checks GitHub every ~2 minutes and rebuilds when there is a new commit
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B clean package -DskipTests'
      }
    }

    stage('Test') {
      steps {
        sh 'mvn -B test'
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Docker Image') {
      steps {
        sh '''
          ARTIFACT=$(ls target/*.war 2>/dev/null | head -1)
          if [ -n "$ARTIFACT" ]; then
            printf 'FROM tomcat:9.0-jdk17\\nRUN rm -rf /usr/local/tomcat/webapps/*\\nCOPY %s /usr/local/tomcat/webapps/ROOT.war\\nEXPOSE 8080\\n' "$ARTIFACT" > Dockerfile.app
          else
            ARTIFACT=$(ls target/*.jar | grep -v original | head -1)
            printf 'FROM eclipse-temurin:17-jre\\nCOPY %s /app.jar\\nEXPOSE 8080\\nENTRYPOINT ["java","-jar","/app.jar"]\\n' "$ARTIFACT" > Dockerfile.app
          fi
          cat Dockerfile.app
          docker build -f Dockerfile.app -t ${IMAGE}:${BUILD_NUMBER} -t ${IMAGE}:latest .
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          docker rm -f onlinebookstore || true
          docker run -d --name onlinebookstore --restart unless-stopped -p ${APP_PORT}:8080 ${IMAGE}:latest
          docker ps --filter name=onlinebookstore
        '''
      }
    }
  }

  post {
    success { echo "Deployed. Open http://<server-public-ip>:${env.APP_PORT}" }
    failure { echo 'Build failed - open the failed stage and read its console log.' }
  }
}
