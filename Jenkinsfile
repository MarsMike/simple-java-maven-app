pipeline {

  /*
   * Run everything on an existing agent configured with a label 'docker'.
   * This agent will need docker, git and a jdk installed at a minimum.
   */
  agent {
    node {
      label 'docker'
    }
  }

  // using the Timestamper plugin we can add timestamps to the console log
  options {
    timestamps()
  }

  environment {
    //Use Pipeline Utility Steps plugin to read information from pom.xml into env variables
    REPO = 'marsmike'
    IMAGE = readMavenPom().getArtifactId()
    VERSION = readMavenPom().getVersion()
  }

  stages {
    stage('Build') {
      agent {
        docker {
          /*
           * Reuse the workspace on the agent defined at top-level of Pipeline but run inside a container.
           * In this case we are running a container with maven so we don't have to install specific versions
           * of maven directly on the agent
           */
          reuseNode true
          image 'maven:3.5.0-jdk-8'
          args '-v /root/.m2:/root/.m2'
        }
      }
      steps {
        // using the Pipeline Maven plugin we can set maven configuration settings, publish test results, and annotate the Jenkins console
        withMaven(options: [findbugsPublisher(), junitPublisher(ignoreAttachments: false)]) {
          sh 'mvn clean findbugs:findbugs package'
        }
      }
      post {
        success {
          // we only worry about archiving the jar file if the build steps are successful
          archiveArtifacts(artifacts: '**/target/*.jar', allowEmptyArchive: true)
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Quality Analysis') {
      parallel {
        // run Sonar Scan and Integration tests in parallel. This syntax requires Declarative Pipeline 1.2 or higher
        stage ('Integration Test') {
          agent any  //run this stage on any available agent
          steps {
            echo 'Run integration tests here...'
          }
        }
        stage('Sonar Scan') {
          agent {
            docker {
              // we can use the same image and workspace as we did previously
              reuseNode true
              image 'maven:3.5.0-jdk-8'
            }
          }
          environment {
            //use 'sonar' credentials scoped only to this stage
            SONAR = credentials('sonar')
          }
          steps {
            sh 'mvn sonar:sonar \
                  -Dsonar.projectKey=simple-java-maven-app \
                  -Dsonar.host.url=https://sonar.kube.im \
                  -Dsonar.login=0b50d46a7587f219dfdad9b705e5e7e75ebe9b9f'
          }
        }
      }
    }

    stage('Build and Publish Image') {
     
      steps {
        /*
         * Multiline strings can be used for larger scripts. It is also possible to put scripts in your shared library
         * and load them with 'libaryResource'
         */
        sh """
          docker build -t ${REPO}/${IMAGE} .
          docker tag ${REPO}/${IMAGE} ${REPO}/${IMAGE}:${VERSION}
          docker push ${REPO}/${IMAGE}:${VERSION}
        """
      }
    }
  }

  post {
    failure {
      // notify users when the Pipeline fails
      mail to: 'mike@objektarium.de',
          subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
          body: "Something is wrong with ${env.BUILD_URL}"
    }
  }
}
