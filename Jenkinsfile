pipeline {
  agent {
    kubernetes {
      yamlFile 'KubernetesPod.yaml'
    }
  }

  parameters {
    choice(
      name: 'PUSH_IMAGE_MODE',
      choices: ['auto', 'true', 'false'],
      description: 'auto = push only for release builds'
    )
  }

  environment {
    FAMILY = 'linux'
    ARCHITECTURE = 'amd64'
  }


  stages {

    stage('prepare') {
      steps {
        container('tools') {
          dir('project') {
            echo "prepare the application (FAMILY=${env.FAMILY}, ARCH=${env.ARCHITECTURE})"
            checkout([
              $class: 'GitSCM',
              branches: [[name: '*/main']],
              userRemoteConfigs: [[url: 'https://github.com/rsmaxwell/diaries.git']],
              extensions: [
                [$class: 'SubmoduleOption',
                  disableSubmodules: false,
                  recursiveSubmodules: true,
                  parentCredentials: true,
                  reference: '',
                  trackingSubmodules: false
                ]
              ]
            ])
            sh('./scripts/prepare.sh')

            script {
              def repository = sh(
                script: ". build/buildinfo && printf '%s' \"\$REPOSITORY\"",
                returnStdout: true
              ).trim()
      
              def mode = params.PUSH_IMAGE_MODE ?: 'auto'
      
              if (mode == 'true') {
                env.EFFECTIVE_PUSH_IMAGE = 'true'
              } else if (mode == 'false') {
                env.EFFECTIVE_PUSH_IMAGE = 'false'
              } else {
                env.EFFECTIVE_PUSH_IMAGE = (repository == 'releases') ? 'true' : 'false'
              }
      
              echo "REPOSITORY=${repository}, PUSH_IMAGE_MODE=${mode}, EFFECTIVE_PUSH_IMAGE=${env.EFFECTIVE_PUSH_IMAGE}"
            }
          }
        }
      }
    }

    stage('build') {
      steps {
        container('node') {
          dir('project/diaries-client') {
            echo 'build the Angular Webapp'

            sh('pwd')
            sh('ls -al')

            sh('./scripts/build.sh')
          }
        }
      }
    }

    stage('image') {
      when {
        expression { env.EFFECTIVE_PUSH_IMAGE == 'true' }
      }
      environment {
        IMAGE_REGISTRY  = 'docker.io'
        IMAGE_NAMESPACE = 'rsmaxwell'
        IMAGE_NAME      = 'diaries-client'
      }
      steps {
        container('buildkitd') {
          dir('project/diaries-client') {
            withCredentials([
              usernamePassword(
                credentialsId: 'docker-registry-creds',
                usernameVariable: 'DOCKER_USERNAME',
                passwordVariable: 'DOCKER_PASSWORD'
              )
            ]) {
              echo 'build docker image and publish to docker repository'
              sh('./scripts/image.sh')
            }
          }
        }
      }
    }
  }
}
