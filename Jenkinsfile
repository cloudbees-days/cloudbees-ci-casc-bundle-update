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
        BUNDLE_ID = event.controller.bundle_id.toString().toLowerCase()        
        GITHUB_ORGANIZATION = event.github.organization.toString().replaceAll(" ", "-")
        GITHUB_REPOSITORY = event.github.repository.toString().toLowerCase()
        GITHUB_USER = event.github.user.toString().toLowerCase(
      }
      when {
        triggeredBy 'EventTriggerCause'
        environment name: 'CONTROLLER_CASC_UPDATE_SECRET', value: OPS_CASC_UPDATE_SECRET
      }
      steps {
        container("kubectl") {
          sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git ."
          sh "mkdir -p ${BUNDLE_ID}"
          sh "find -name '*.yaml' | xargs cp --parents -t ${BUNDLE_ID}"
          sh "kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_home/jcasc-bundles-store/ -c jenkins"
        }
      }
    }
  }
}
