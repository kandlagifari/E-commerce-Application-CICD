pipeline {
  agent any
  tools {
    maven 'maven'
    jdk 'jdk17'
  }
  environment {
    SCANNER_HOME = tool 'sonarqube-scanner'
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
    stage('SonarQube Analysis') {
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
    stage('Docker Build and Tag'){
      steps{
        script{
          withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
            sh "docker build -t kandlagifari/ecommerce-application:latest -f docker/Dockerfile ."
          }
        } 
      }
    }
    stage('Trivy Image Scan'){
      steps{
        sh "trivy image kandlagifari/ecommerce-application:latest > trivy-report.txt"
      }
    }
    stage('Push Image to Dockerhub'){
      steps{
        script{
          withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
            sh "docker push kandlagifari/ecommerce-application:latest"
          }
        } 
      }
    }
    stage('Deploy to Kubernetes'){
      steps{
        script{
          withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s-service-account-token', namespace: 'ecommerce', serverUrl: 'https://kind-control-plane:6443']]) {
            sh "kubectl apply -f deploymentservice.yaml -n ecommerce --validate=false"
            sh "kubectl get svc -n ecommerce"
          }
        } 
      }
    }
  }
}
