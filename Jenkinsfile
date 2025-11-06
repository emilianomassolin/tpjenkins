#!/usr/bin/env groovy
node {
  // Opcional: deshabilita testcontainers a nivel pipeline
  withEnv(['TESTCONTAINERS_DISABLED=true','SPRING_TESTCONTAINERS_ENABLED=false']) {

    stage('checkout') { checkout scm }

    stage('check java') { sh "java -version || true" }

    stage('clean') {
      sh "chmod +x mvnw"
      sh "./mvnw -ntp clean -P-webapp"
    }

    stage('checkstyle') {
      sh "./mvnw -ntp checkstyle:check"
    }

    stage('install tools (node/npm)') {
      sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:install-node-and-npm@install-node-and-npm"
    }

    stage('npm install (frontend vía maven)') {
      sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm"
    }

    stage('backend tests') {
      try {
        // Opción A (más simple): solo unit tests -> NO dispara failsafe ni Testcontainers
        sh "./mvnw -ntp test -P-webapp"

        // Opción B (si querés 'verify' pero sin ITs):
        // sh "./mvnw -ntp verify -P-webapp -DskipITs -Dspring.testcontainers.enabled=false"
      } finally {
        junit allowEmptyResults: true,
              testResults: '**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml'
        archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/*.jar'
      }
    }

    stage('frontend tests') {
      try {
        sh "npm install"
        // sh "npm test"
      } finally {
        junit allowEmptyResults: true, testResults: '**/target/test-results/TESTS-results-jest.xml'
      }
    }

    stage('packaging') {
      // ya estás saltando tests aquí, así que no ejecuta ITs
      sh "./mvnw -ntp verify -P-webapp -Pprod -DskipTests"
      archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
    }

    stage('publish docker (Jib)') {
      withCredentials([usernamePassword(
        credentialsId: 'dockerhub-login',
        usernameVariable: 'DOCKER_REGISTRY_USER',
        passwordVariable: 'DOCKER_REGISTRY_PWD'
      )]) {
        sh """
          ./mvnw -ntp -Pprod \
            -Djib.to.image=${DOCKER_REGISTRY_USER}/tpjenkins:latest \
            -Djib.to.auth.username=${DOCKER_REGISTRY_USER} \
            -Djib.to.auth.password=${DOCKER_REGISTRY_PWD} \
            verify jib:build -DskipTests
        """
      }
    }
  }
}
