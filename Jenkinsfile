#!/usr/bin/env groovy

REPOSITORY = 'router'

node ('mongodb-2.4') {
  env.REPO      = 'alphagov/router'
  env.BUILD_DIR = '__build'
  env.GOPATH    = "${WORKSPACE}/${BUILD_DIR}"
  env.SRC_PATH  = "${env.GOPATH}/src/github.com/${REPO}"

    def govuk = load '/var/lib/jenkins/groovy_scripts/govuk_jenkinslib.groovy'

  try {
    stage("Checkout") {
      checkout scm
    }

    stage("Setup build environment") {
      // Clean GOPATH: Recursively delete everything in the current directory
      dir(env.GOPATH) {
        deleteDir()

        // Create Build-Path
        sh "mkdir -p ${env.SRC_PATH}"
    }

      // Seed Build-Path
      dir(env.WORKSPACE) {
        sh "/usr/bin/rsync -a ./ ${env.SRC_PATH} --exclude=$BUILD_DIR"
      }
    }

    // Start Build
    stage("Build") {
      dir(env.SRC_PATH) {
        sh 'BINARY=$WORKSPACE/router make clean build'
      }
    }

    // Run tests
    wrap([$class: 'AnsiColorBuildWrapper']) {
      stage("Test") {
        dir(env.SRC_PATH) {
          sh 'BINARY=$WORKSPACE/router make test'

          sh '$WORKSPACE/router -version'
        }
      }
    }

    // Archive Binaries from build
    stage("Archive Artifact") {
      archiveArtifacts 'router'
    }
  
    stage("publish to s3") {
      step([$class: 'S3BucketPublisher', entries: [[
        sourceFile: 'router',
        bucket: 'govuk-router-builds',
        selectedRegion: 'eu-west-1',
        noUploadOnFailure: true,
        managedArtifacts: true,
        flatten: true,
        showDirectlyInBrowser: true,
        keepForever: true,]],
        profileName: 'govuk-router',
        dontWaitForConcurrentBuildCompletion: false,
      ])
    }

    // Update GitHub Status
    stage("Push release tag") {
      echo 'Pushing tag'
      govuk.pushTag(REPOSITORY, env.BRANCH_NAME, 'release_' + env.BUILD_NUMBER)
    }

    // Deploy application
    stage("Deploy") {
      govuk.deployIntegration(REPOSITORY, env.BRANCH_NAME, 'release', 'deploy')
    }

  } catch (e) {
      currentBuild.result = "FAILED"
      step([$class: 'Mailer',
            notifyEveryUnstableBuild: true,
            recipients: 'govuk-ci-notifications@digital.cabinet-office.gov.uk',
            sendToIndividuals: true])
    throw e
    }

}
