pipeline {
    agent any
    environment {
      ORG               = 'threem'
      APP_NAME          = 'aspnet-app'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          sh "docker build -t \$DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION ."
          sh "docker push \$DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"

          dir ('./charts/preview') {
            sh "make preview"
            sh "jx preview --app $APP_NAME --dir ../.."
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          git 'https://github.com/threem/aspnet-app.git'
          // so we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"
          dir ('./charts/aspnet-app') {
            sh "make tag"
          }
          sh "docker build -t \$DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION) ."
          sh "docker push \$DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/aspnet-app') {
            sh 'jx step changelog --version v\$(cat ../../VERSION)'

            // release the helm chart
            sh 'make release'

            // promote through all 'Auto' promotion Environments
            sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION) --no-wait'
          }
        }
      }
    }
  }
