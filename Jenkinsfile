#!/usr/bin/env groovy

node {
  stage('checkout') {
    checkout scm
  }

  // Construcción y tests dentro de la imagen de JHipster (trae Maven + Node)
  docker.image('jhipster/jhipster:v8.11.0')
        .inside('-u jhipster -e MAVEN_OPTS="-Duser.home=./"') {

    stage('check java') {
      sh "java -version"
    }

    stage('clean') {
      sh "chmod +x mvnw"
      sh "./mvnw -ntp clean -P-webapp"
    }

    stage('checkstyle') {
      sh "./mvnw -ntp checkstyle:check"
    }

    stage('install tools') {
      sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:install-node-and-npm@install-node-and-npm"
    }

    stage('npm install (frontend)') {
      sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm"
    }

    stage('backend tests') {
      try {
        sh "./mvnw -ntp verify -P-webapp"
      } finally {
        junit allowEmptyResults: true,
              testResults: '**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml'
        archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/*.jar'
      }
    }

    // ⚠️ Para que este stage reporte a JUnit,
    // tu "npm test" debe generar junit.xml (por ejemplo con jest-junit).
    stage('frontend tests') {
      try {
        sh "npm install"
        // Si usás Jest, asegurate de configurar jest-junit y exportar JEST_JUNIT_OUTPUT
        // sh "npm test"
      } finally {
        junit allowEmptyResults: true, testResults: '**/target/test-results/TESTS-results-jest.xml'
      }
    }

    stage('packaging') {
      sh "./mvnw -ntp verify -P-webapp -Pprod -DskipTests"
      archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
    }
  }

  // Publicar imagen con Jib a Docker Hub en un solo stage
  stage('publish docker') {
    withCredentials([usernamePassword(
      credentialsId: 'dockerhub-login',
      usernameVariable: 'DOCKER_REGISTRY_USER',
      passwordVariable: 'DOCKER_REGISTRY_PWD'
    )]) {
      // Ajustá el nombre del repo/imagen a lo que uses en Docker Hub
      sh """
        ./mvnw -ntp -Pprod \
          -Djib.to.image=${DOCKER_REGISTRY_USER}/tpjenkins:latest \
          -Djib.to.auth.username=${DOCKER_REGISTRY_USER} \
          -Djib.to.auth.password=${DOCKER_REGISTRY_PWD} \
          verify jib:build
      """
    }
  }
}
