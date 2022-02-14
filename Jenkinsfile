def event = currentBuild.getBuildCauses()[0].event
library 'pipeline-library'
pipeline {
  agent none
  environment {
    OPS_CASC_UPDATE_SECRET = credentials('casc-update-secret')
    CONTROLLER_CASC_UPDATE_SECRET = event.secret.toString()
  }
  options { timeout(time: 10, unit: 'MINUTES') }
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
        BUNDLE_ID = "${CONTROLLER_FOLDER}-${GITHUB_REPOSITORY}"
        AUTO_RELOAD = event.casc.auto_reload.toString()
      }
      when {
        triggeredBy 'EventTriggerCause'
        environment name: 'CONTROLLER_CASC_UPDATE_SECRET', value: OPS_CASC_UPDATE_SECRET
      }
      stages {
        stage('Copy Files CI Workshop') {
          when {
            environment name: 'GITHUB_REPOSITORY', value: 'ci-config-bundle'
          }
          environment {
            BUNDLE_ID = event.controller.bundle_id.toString().toLowerCase()  
          }
          steps {
            container("kubectl") {
              sh "rm -rf ./${BUNDLE_ID} || true"
              sh "mkdir -p ${BUNDLE_ID}"
              sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git ${BUNDLE_ID}"
              sh "kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_home/jcasc-bundles-store/ -c jenkins"
            }
          }
        }
        stage('Copy Files CasC Workshop') {
          when {
            not {
              environment name: 'GITHUB_REPOSITORY', value: 'ci-config-bundle'
            }
          }
          steps {
            container("kubectl") {
              sh "rm -rf ./${BUNDLE_ID} || true"
              sh "mkdir -p ${BUNDLE_ID}"
              sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git ${BUNDLE_ID}"
              sh "kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_home/jcasc-bundles-store/ -c jenkins"
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
