pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "hsjarbin/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://192.168.24.30"
    applicationURI = "/increment/99"
  }

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later AA
            }
        }
      stage('Unit Tests - JUnit and JaCoCo') {
            steps {
	            sh "mvn test"
	          }
	      }

      stage('Mutation Tests - PIT') {
        steps {
          sh "mvn org.pitest:pitest-maven:mutationCoverage"
        }
      }

      stage('SonarQube - SAST') {
        steps {
          withSonarQubeEnv('SonarQube') {
            sh "mvn sonar:sonar \
              -Dsonar.projectKey=numeric-application \
              -Dsonar.host.url=http://192.168.24.30:9000"
          }
          timeout(time: 2, unit: 'MINUTES') {
            script {
              waitForQualityGate abortPipeline: true
            }
          }
        }
      }

      stage('Vulnerability Scan - Docker ') {
        steps {
          parallel (
            "Dependency Scan": {
              sh "mvn dependency-check:check"
            },
            "Trivy Scan": {
              sh "bash trivy-docker-image-scan.sh"
            },
            "OPA Conftest": {
              sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
            }
          )
        }
      }

      stage('Docker Build and Push') {
          steps {
            sh 'id -a' // checking jenkins user
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'sudo docker build -t hsjarbin/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push hsjarbin/numeric-app:""$GIT_COMMIT""'
            }
          }
  	  }

      stage('Vulnerability Scan - Kubernetes') {
        steps {
          parallel (
            "OPA Scan": {
              sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
            },
            "Kubesec Scan": {
              sh "bash kubesec-scan.sh"
            },
            "Trivy SCan": {
              sh "bash trivy-k8s-scan.sh"
            }
          )
        }
      }

      stage('K8S Update Deployment - DEV') {
        steps {
          withKubeConfig([credentialsId: 'kubeconfig']) {
            sh "bash k8s-deployment.sh"
          }
        }
      }

      stage('K8S Check Status Deployment - DEV') {
        steps {
          withKubeConfig([credentialsId: 'kubeconfig']) {
            sh "bash k8s-deployment-rollout-status.sh"
          }
        }
      }

      stage('Integration Tests - DEV') {
        steps {
          script {
            try {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash integration-test.sh"
              }
            }
            catch(e) {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "kubectl -n default rollout undo deploy ${deploymentName}"
              }
              throw e
            }
          }
        }
      }

      stage('OWASP ZAP - DAST') {
        steps {
          withKubeConfig([credentialsId: 'kubeconfig']) {
            sh 'bash zap.sh'
            }
          }
        }
      }

    }

    post {
      always {
        junit 'target/surefire-reports/*.xml'
        jacoco execPattern: 'target/jacoco.exec'
        pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      }
      // success {
      //
      // }
      // failure {
      //
      // }
    }
}
