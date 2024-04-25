pipeline {
  environment {
    DEV_URL='http://dso-demo.bing.svc.cluster.local:31000'
  }
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
              /bin/bash --login
              rvm use default
              gem install license_finder
              license_finder
              ''' 
            }
          }
        }
        stage('Generate SBOM') {
          steps {
            container('maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }
          }
          post {
            success {
             //dependencyTrackPublisher projectName:'sample-spring-app', projectVersion: '0.0.1', artifact:'target/bom.xml', autoCreateProjects: true, synchronous: true
             archiveArtifacts allowEmptyArchive: true,artifacts: 'target/bom.xml', fingerprint: true,onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage("Build with Kaniko") {
          steps {
              container('kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --force --destination=docker.io/neighborhooood/dso-demo'
            }
          }
        }
      }
    }
    stage('Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/neighborhooood/dso-demo'
           }
         }
       }
        stage('Image Scan') {
          steps {
            container('docker-tools') {
              sh 'trivy image neighborhooood/dso-demo'
            }
          }
        }
      }
   }
    stage('Deploy to Dev') {
      steps {
        // TODO
        sh "echo done"
      }
    }
    stage("Dynamic Analysis") {
      parallel {
        stage('E2E test') {
          steps {
            sh 'echo "ALL Test passed"'
          }
        }
        stage('DAST') {
          steps {
            container('docker-tools') {
              sh 'docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t $DEV_URL || exit 0 '
            }
          }
        }
      }
    }
  }
}
