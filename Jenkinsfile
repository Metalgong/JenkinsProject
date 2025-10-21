pipeline {
  agent any

  tools {
    jdk "JDK17"
    maven "MAVEN3.9"
  }

  environment {
    // ===== Nexus Repository Info , ok =====
    NEXUS_VERSION       = "nexus3"
    NEXUS_PROTOCOL      = "http"
    NEXUS_URL           = "3.88.202.19:8081"
    NEXUS_REPOSITORY    = "vprofile-release"
    NEXUS_CREDENTIAL_ID = "nexus-creds"
    ARTVERSION          = "${env.BUILD_ID}"

    // ===== Slack Channel =====
    SLACK_CHANNEL       = '#all-vprofileapp-team'

    // ===== AWS Deployment Info =====
    AWS_REGION          = "us-east-1"
    AWS_ACCOUNT_ID      = "686395477549"
    ECR_REPOSITORY      = "vprofileapp"
    ECS_CLUSTER         = "vprofile-cluster"
    ECS_SERVICE         = "vprofile-service"
    AWS_CREDENTIALS_ID  = "aws-creds"
  }

  stages {

    // ======================== STEP 1: Verify Tool Setup ========================
    stage('Tool Setup') {
      steps {
        echo "🔧 Verifying tool installations..."
        sh 'mvn -version'
        sh 'java -version'
      }
    }

    // ======================== STEP 2: Checkout Source Code =====================
    stage('Checkout Source Code') {
      steps {
        slackSend(channel: env.SLACK_CHANNEL,
                  message: "🟡 Build #${BUILD_NUMBER} started for *${env.JOB_NAME}* by ${env.BUILD_USER ?: 'Jenkins'}",
                  color: '#FFFF00')
        git url: 'https://github.com/hkhcoder/vprofile-project.git', branch: 'atom'
      }
    }

    // ======================== STEP 3: Build the Application ====================
    stage('Build Application') {
      steps {
        echo "🏗️ Building Maven project..."
        sh 'mvn clean install -DskipTests'
      }
    }

    // ======================== STEP 4: Run Unit Tests ===========================
    stage('Run Unit Tests') {
      steps {
        echo "🧪 Running unit tests..."
        sh 'mvn test'
      }
    }

    // ======================== STEP 5: Run Integration Tests ====================
    stage('Run Integration Tests') {
      steps {
        echo "🔍 Running integration tests..."
        sh 'mvn verify -DskipUnitTests'
      }
    }

    // ======================== STEP 6: Code Analysis - Checkstyle ===============
    stage('Code Analysis - Checkstyle') {
      steps {
        echo "📋 Performing Checkstyle analysis..."
        sh 'mvn checkstyle:checkstyle'
      }
    }

    // ======================== STEP 7: Code Analysis - SonarQube ================
    stage('Code Analysis - SonarQube') {
      environment {
        scannerHome = tool 'sonarscanner4'
      }
      steps {
        script {
          echo "🧠 Running SonarQube analysis..."
          withSonarQubeEnv('sonar-pro') {
            sh '''
              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=vprofile \
                -Dsonar.projectName=vprofile-repo \
                -Dsonar.projectVersion=1.0 \
                -Dsonar.sources=src/ \
                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
            '''
          }
          def qg = waitForQualityGate()
          if (qg.status != 'OK') {
            error "❌ SonarQube Quality Gate failed!"
          }
          echo "✅ SonarQube Quality Gate passed successfully."
        }
      }
    }

    // ======================== STEP 8: Publish to Nexus =========================
    stage('Publish Artifact to Nexus') {
      when {
        expression {
          sh(returnStdout: true, script: "git rev-parse --abbrev-ref HEAD").trim() == 'atom'
        }
      }
      steps {
        script {
          def pom = readMavenPom file: "pom.xml"
          def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
          def artifactPath = filesByGlob[0].path

          echo "📦 Uploading artifact ${pom.artifactId}-${pom.version}.war to Nexus..."
          nexusArtifactUploader(
            nexusVersion:  NEXUS_VERSION,
            protocol:      NEXUS_PROTOCOL,
            nexusUrl:      NEXUS_URL,
            groupId:       pom.groupId,
            version:       ARTVERSION,
            repository:    NEXUS_REPOSITORY,
            credentialsId: NEXUS_CREDENTIAL_ID,
            artifacts: [
              [ artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging ],
              [ artifactId: pom.artifactId, classifier: '', file: "pom.xml", type: "pom" ]
            ]
          )
        }
      }
    }

    // ======================== STEP 9: Create Dockerfile (Tomcat 10) ===========
    stage('Create Dockerfile') {
      steps {
        script {
          echo "🧱 Creating Dockerfile for Tomcat 10 deployment..."
          sh '''
            cat > Dockerfile <<'EOF'
FROM tomcat:10-jdk11-temurin
LABEL maintainer="vprofile-pipeline"
COPY target/*.war /usr/local/tomcat/webapps/vprofile.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
EOF
          '''
          echo "✅ Tomcat 8.5 Dockerfile created successfully."
          sh 'cat Dockerfile'
        }
      }
    }

    // ======================== STEP 10: Build Docker Image ======================
    stage('Build Docker Image') {
      steps {
        script {
          echo "🛠️ Building Docker image..."
          sh '''
            docker build -t ${ECR_REPOSITORY}:${BUILD_NUMBER} .
            docker tag ${ECR_REPOSITORY}:${BUILD_NUMBER} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${BUILD_NUMBER}
          '''
        }
      }
    }

    // ======================== STEP 11: Push Docker Image to ECR ================
    stage('Push Docker Image to AWS ECR') {
      steps {
        script {
          echo "📤 Pushing Docker image to AWS ECR..."
          withAWS(credentials: env.AWS_CREDENTIALS_ID, region: env.AWS_REGION) {
            sh '''
              aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
              docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${BUILD_NUMBER}
            '''
          }
          echo "✅ Docker image pushed to ECR successfully."
        }
      }
    }

    // ======================== STEP 12: Deploy to AWS ECS =======================
    stage('Deploy to AWS ECS') {
      steps {
        script {
          echo "🚀 Deploying new image to ECS..."
          withAWS(credentials: env.AWS_CREDENTIALS_ID, region: env.AWS_REGION) {
            sh '''
              aws ecs update-service \
                --cluster ${ECS_CLUSTER} \
                --service ${ECS_SERVICE} \
                --force-new-deployment \
                --region ${AWS_REGION}
            '''
          }
          echo "✅ ECS deployment triggered successfully."
        }
      }
    }
  }

  post {
    success {
      script {
        echo "✅ Build and Deployment succeeded!"
        slackSend(channel: env.SLACK_CHANNEL,
                  message: "✅ *Build #${BUILD_NUMBER}* for *${env.JOB_NAME}* succeeded and deployed to ECS! 🚀",
                  color: '#00FF00')
      }
    }
    failure {
      script {
        echo "❌ Build or Deployment failed!"
        slackSend(channel: env.SLACK_CHANNEL,
                  message: "❌ *Build #${BUILD_NUMBER}* for *${env.JOB_NAME}* failed! Check Jenkins for details.",
                  color: '#FF0000')
      }
    }
  }
}
