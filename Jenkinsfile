pipeline {
  agent any
  tools {
    maven 'maven'
    jdk 'jdk17'
  }
  environment {
    SCANNER_HOME = tool 'sonar-scanner'
  }
  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/kandlagifari/E-commerce-Application-CICD.git'
      }
    }
    stage('Code Compilation') {
      steps {
        sh "mvn compile"
      }
    }
    stage('Unit Tests') {
      steps {
        sh "mvn test -DskipTests=true"
      }
    }
    stage('SonaeQube Analysis') {
      steps {
        withSonarQubeEnv('sonar') {
          sh ''' 
          $SCANNER_HOME/bin/sonar-scanner \
          -Dsonar.projectKey=E-commerce-Application \
          -Dsonar.projectName=E-commerce-Application \
          -Dsonar.java.binaries=. 
          '''
        }
      }
    }
    stage('OWASP Dependency-Check') {
      steps {
        dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dependency-check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }
    stage('Maven Build') {
      steps {
        sh "mvn package -DskipTests=true"
      }
    }
    stage('Deploy Artifact to Nexus') {
      steps {
        withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
          sh "mvn deploy -DskipTests=true"
        }
      }
    }
  }
}
