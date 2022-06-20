def event = currentBuild.getBuildCauses()[0].event
library 'pipeline-library'
pipeline {
  agent none
  environment {
    OPS_CASC_UPDATE_SECRET = credentials('casc-update-secret')
    CONTROLLER_CASC_UPDATE_SECRET = event.secret.toString()
  }
  options { timeout(time: 4, unit: 'MINUTES') }
  triggers {
    eventTrigger jmespathQuery("controller.action=='casc_bundle_update'")
  }
  stages {
    stage('Configuration Bundle Update') {
      agent {
        kubernetes {
          yaml libraryResource ('podtemplates/kubectl.yml')
        }
      }
      environment {       
        GITHUB_ORGANIZATION = event.github.organization.toString().replaceAll(" ", "-")
        GITHUB_REPOSITORY = event.github.repository.toString().toLowerCase()
        CONTROLLER_FOLDER = GITHUB_ORGANIZATION.toLowerCase()
        BUNDLE_ID = event.controller.bundle_id.toString().toLowerCase()  
        AUTO_RELOAD = event.casc.auto_reload.toString()
      }
      when {
        triggeredBy 'EventTriggerCause'
        environment name: 'CONTROLLER_CASC_UPDATE_SECRET', value: OPS_CASC_UPDATE_SECRET
      }
      stages {
        stage('Bundle Update') {
          stages {
            stage('Copy Files') {
              steps {
                container("kubectl") {
                  sh '''
                    rm -rf ./${BUNDLE_ID} || true
                    rm -rf ./checkout || true
                    mkdir -p ${BUNDLE_ID}
                    mkdir -p checkout
                    git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git checkout
                  '''
                  dir('checkout/bundle') {
                    sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
                  }
                  withCredentials([usernamePassword(credentialsId: "cloudbees-ci-casc-workshop-github-app",
                                          usernameVariable: 'GITHUB_APP',
                                          passwordVariable: 'GITHUB_ACCESS_TOKEN')]) {
                    sh '''
                      ls -la ${BUNDLE_ID}
                      mkdir -p workshop-casc-bundles
                      cd workshop-casc-bundles
                      git init
                      git config user.email "beedemo-dev@workshop.cb-sa.io"
                      git config user.name "cloudbees-days"
                      git config pull.rebase false
                      git remote add origin https://x-access-token:${GITHUB_ACCESS_TOKEN}@github.com/cloudbees-days/workshop-casc-bundles.git
                      git pull origin main
                      git checkout main
                      rm -rf ${BUNDLE_ID} || true
                      cp ../${BUNDLE_ID} .
                      git add *
                      git commit -a -m "updating bundle ${BUNDLE_ID}"
                      git push origin main
                    '''
                  }
                  
                  withCredentials([usernamePassword(credentialsId: 'admin-cli-token', usernameVariable: 'JENKINS_CLI_USR', passwordVariable: 'JENKINS_CLI_PSW')]) {
                    sh  '''
                      curl --user "$JENKINS_CLI_USR:$JENKINS_CLI_PSW" -XPOST \
                        http://cjoc/cjoc/load-casc-bundles/checkout
                    '''
                  }
                }
              }              
            }
            stage('Auto Reload Bundle') {
              when {
                environment name: 'AUTO_RELOAD', value: "true"
              }
              steps {
                echo "begin config bundle reload"
                withCredentials([usernamePassword(credentialsId: 'admin-cli-token', usernameVariable: 'JENKINS_CLI_USR', passwordVariable: 'JENKINS_CLI_PSW')]) {
                  sh '''
                    curl --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XGET http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/check-bundle-update
                    curl --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XPOST http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/reload-bundle
                  '''
                }
              }
            }
          }
        }
      }  
    }
  }
}
