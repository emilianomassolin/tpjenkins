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
    // Solo unit tests: no empaqueta ni corre ITs
    sh "./mvnw -ntp test -P-webapp"
  } finally {
    junit allowEmptyResults: true,
          testResults: '**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml'
    // Quita el archiveArtifacts de acá (todavía no existe el jar)
  }
}

stage('packaging') {
  // Empaqueta sin tests (ya corrimos los unit tests antes)
  sh "./mvnw -ntp verify -P-webapp -Pprod -DskipTests"
  archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: false
}

stage('frontend tests') {
  try {
    // Instalar dependencias con el plugin (equivale a "npm ci" o "npm install")
    sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm -Dfrontend.npm.arguments='ci --no-audit --no-fund'"

    // Si tenés tests en package.json ("test"), ejecutalos así:
    // sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm -Dfrontend.npm.arguments='test -- --ci'"

  } finally {
    // Si querés reportes JUnit de frontend, configurá jest-junit y apuntá al XML correcto.
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
