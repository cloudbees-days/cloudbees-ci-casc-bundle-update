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
                    sed -i "s/REPLACE_REPO/$GITHUB_REPOSITORY/g" checkout/bundle/bundle.yaml
                  '''
                  dir('checkout/bundle') {
                    sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
                  }
                  sh '''
                    ls -la ${BUNDLE_ID}
                    kubectl exec cjoc-0 -c jenkins -- rm -rf /var/jenkins_config/jcasc-bundles-store/${BUNDLE_ID}/
                    kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_config/jcasc-bundles-store/ -c jenkins
                  '''
                  
                  withCredentials([usernamePassword(credentialsId: 'admin-cli-token', usernameVariable: 'JENKINS_CLI_USR', passwordVariable: 'JENKINS_CLI_PSW')]) {
                    sh  '''
                      curl --user "$JENKINS_CLI_USR:$JENKINS_CLI_PSW" -XPOST \
                        -H "Accept: application/json"  \
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
                    curl --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XGET -H "Accept: application/json" http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/check-bundle-update
                    curl --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XPOST -H "Accept: application/json" http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/reload-bundle
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
