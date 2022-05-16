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
        stage('CI Workshop') {
          when {
            environment name: 'GITHUB_REPOSITORY', value: 'cloudbees-ci-config-bundle'
          }
          environment {
            BUNDLE_ID = event.controller.bundle_id.toString().toLowerCase()  
          }
          stages {
            stage('Copy Files') {
              steps {
                container("kubectl") {
                  sh "rm -rf ./${BUNDLE_ID} || true"
                  sh "rm -rf ./checkout || true"
                  sh "mkdir -p ${BUNDLE_ID}"
                  sh "mkdir -p checkout"
                  sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git checkout"
                  dir('checkout/bundle') {
                    sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
                  }
                  sh "ls -la ${BUNDLE_ID}"
                  sh "kubectl exec cjoc-0 -c jenkins -- rm -rf /var/jenkins_home/jcasc-bundles-store/${BUNDLE_ID}/"
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
                  waitUntil {
                    script {
                      def UPDATE_AVAILABLE = sh (script: '''curl -s --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XGET http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/check-bundle-update  | jq '.["update-available"]' | tr -d "\n" ''', 
                        returnStdout: true) 
                      echo "update available: ${UPDATE_AVAILABLE}"
                      return (UPDATE_AVAILABLE=="true")
                    }
                  }
                  sh '''                      
                    curl --user $JENKINS_CLI_USR:$JENKINS_CLI_PSW -XPOST http://${BUNDLE_ID}.controllers.svc.cluster.local/${BUNDLE_ID}/casc-bundle-mgnt/reload-bundle
                  '''
                }
              }
            }
          }
        }
        stage('CasC Workshop') {
          when {
            not {
              environment name: 'GITHUB_REPOSITORY', value: 'cloudbees-ci-config-bundle'
            }
          }
          stages {
            stage('Copy Files') {
              steps {
                container("kubectl") {
                  sh "rm -rf ./${BUNDLE_ID} || true"
                  sh "rm -rf ./checkout || true"
                  sh "mkdir -p ${BUNDLE_ID}"
                  sh "mkdir -p checkout"
                  sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git checkout"
                  dir('checkout/bundle') {
                    sh "cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}/"
                  }
                  sh "ls -la ${BUNDLE_ID}"
                  sh "kubectl exec cjoc-0 -c jenkins -- rm -rf /var/jenkins_home/jcasc-bundles-store/${BUNDLE_ID}/"
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
                      sleep 7
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
