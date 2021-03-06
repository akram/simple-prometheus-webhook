def template = 'https://raw.githubusercontent.com/akram/image-scanning-signing-service/master/examples/image-signing-request-template.yml'

openshift.withCluster() {
  env.NAMESPACE =  openshift.project()
  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?${NAMESPACE}-?/, '').replaceAll(/-?pipeline-?/, '').replace("/", "")
}

pipeline {
  agent { label 'nodejs' }
  stages {
    stage('Code Checkout') {
      steps {
        git url: "${SOURCE_CODE_URL}", branch: "${SOURCE_CODE_REF}"
      }
    }

    stage('Code Build') {
      steps {
       //   sh "/usr/local/s2i/"
        echo "toto"
      }
    }

    stage('Image Build') {
     steps {
        echo 'Building Image'
     /*    sh """
          set +x
          rm -rf oc-build && mkdir -p oc-build/deployments
          for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
            cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
          done
        """
       */
        script {
          openshift.withCluster() {
            echo "Building  ${APP_NAME} from-dir=oc-build"
            build = openshift.startBuild("${APP_NAME}")

            timeout(10) {
              build.untilEach {
                def phase = it.object().status.phase
                echo "Build Status: ${phase}"

                if (phase == "Complete") {
                  return true
                }
                else if (phase == "Failed") {
                  currentBuild.result = "FAILURE"
                  buildErrorLog = it.logs().actions[0].err
                  return true
                }
                else {
                  return false
                }
              }
            }

            if (currentBuild.result == 'FAILURE') {
              error(buildErrorLog)
              return
            }
          }
        }
      }
    }

    stage('Sign Image') {
      steps {
        script {
        def obj = "${APP_NAME}-${env.BUILD_NUMBER}"

          openshift.withCluster() {
            def created = openshift.create(openshift.process(template, "-p IMAGE_SIGNING_REQUEST_NAME=${obj} -p IMAGE_STREAM_TAG=${APP_NAME}:${BUILD_APP_TAG}"))

            def imagesigningrequest = created.narrow('imagesigningrequest').name();

            echo "ImageSigningRequest ${imagesigningrequest.split('/')[1]} Created"

            timeout(time: 10, unit: 'MINUTES') {

              waitUntil() {

                def isr = openshift.selector("${imagesigningrequest}")

                if(isr.object().status) {

                  def phase = isr.object().status.phase

                  if(phase == "Failed") {
                    echo "Signing Action Failed: ${isr.object().status.message}"
                    currentBuild.result = "FAILURE"
                    return true
                  }
                  else if(phase == "Completed") {
                    env.SIGNED_IMAGE = isr.object().status.signedImage
                    echo "Signing Action Completed. Signed Image: ${SIGNED_IMAGE}"
                    return true
                  }
                }
                else {
                  echo "Status is null"
                }

                return false

              }
            }
          }
        }
      }
    }

    stage('Tag Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("${APP_NAME}@${SIGNED_IMAGE}", "${APP_NAME}:${DEPLOY_APP_TAG}")
          }
        }
      }
    }

    stage('Rollout Application') {
      steps {
        script {
          openshift.withCluster() {
            def result = null
            deploymentConfig = openshift.selector("dc", "${APP_NAME}")
            deploymentConfig.rollout().latest()

            timeout(10) {
              result = deploymentConfig.rollout().status("-w")
            }

            if (result.status != 0) {
              error(result.err)
            }
          }
        }
      }
    }
  }
}
