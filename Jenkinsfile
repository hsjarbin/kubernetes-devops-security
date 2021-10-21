pipeline {
  agent any

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later AA
            }
        }
      stage('Unit Tests - JUnit and Jacoco') {
            steps {
	            sh "mvn test"
	          }
            post {
              always {
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
              }
            }
	      }
        stage('Docker Build and Push') {
              steps {
                sh 'id -a' // checking jenkins user
                withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
                  sh 'printenv'
                  sh 'docker build -t hsjarbin/numeric-app:""$GIT_COMMIT"" .'
                  sh 'docker push hsjarbin/numeric-app:""$GIT_COMMIT""'
                }
              }
  	      }
    }
}
