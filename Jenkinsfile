pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  parameters {
    string(name: 'DOCKER_REGISTRY', defaultValue: 'docker.io', description: 'e.g., docker.io or private registry host')
    string(name: 'DOCKER_NAMESPACE', defaultValue: 'youruser', description: 'Docker Hub user/org or registry namespace')
    string(name: 'IMAGE_NAME', defaultValue: 'java-app', description: 'Image repository name')
    string(name: 'IMAGE_TAG', defaultValue: '${BUILD_NUMBER}', description: 'Image tag')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip Unit Tests')
  }

  environment {
    MAVEN_HOME = tool name: 'Maven 3.9', type: 'maven'
    JAVA_HOME  = tool name: 'JDK 17',    type: 'hudson.model.JDK'
    PATH = "${JAVA_HOME}/bin:${MAVEN_HOME}/bin:${env.PATH}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'java -version && mvn -v'
      }
    }

    stage('Build & Test (Parallel)') {
      parallel {
        stage('Build JAR') {
          steps {
            sh 'mvn -B -DskipTests=${SKIP_TESTS} clean package'
          }
          post {
            always { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true }
          }
        }

        stage('Unit Tests') {
          when { expression { return !params.SKIP_TESTS } }
          steps {
            sh 'mvn -B test'
          }
          post {
            always { junit 'target/surefire-reports/*.xml' }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def regPath = (params.DOCKER_REGISTRY == 'docker.io') ? '' : "${params.DOCKER_REGISTRY}/"
          env.FULL_IMAGE = "${regPath}${params.DOCKER_NAMESPACE}/${params.IMAGE_NAME}:${params.IMAGE_TAG}"
        }
        sh 'echo Building image ${FULL_IMAGE}'
        sh 'docker build -t ${FULL_IMAGE} .'
      }
    }

    stage('Docker Login') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo "$DOCKER_PASS" | docker login ${params.DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin'
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        sh 'docker push ${FULL_IMAGE}'
        script {
          def latest = (params.DOCKER_REGISTRY == 'docker.io') ? '' : "${params.DOCKER_REGISTRY}/"
          env.LATEST_IMAGE = "${latest}${params.DOCKER_NAMESPACE}/${params.IMAGE_NAME}:latest"
        }
        sh 'docker tag ${FULL_IMAGE} ${LATEST_IMAGE}'
        sh 'docker push ${LATEST_IMAGE}'
      }
    }
  }

post {
    always {
        node {
            sh 'docker rmi my-image || true'
        }
    }
}

post {
    always {
        node {
            sh 'docker rmi my-image || true'
        }
    }
}

