/**
 * The openstack-ardana-gerrit Jenkins Pipeline
 */

pipeline {
  // skip the default checkout, because we want to use a custom path
  options {
    skipDefaultCheckout()
    timestamps()
  }

  agent {
    node {
      label 'cloud-pipeline'
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {

    stage('Setup workspace') {
      steps {
        script {
          currentBuild.displayName = "#${BUILD_NUMBER}: ${gerrit_change_ids}"
        }
        sh('''
          git clone $git_automation_repo --branch $git_automation_branch automation-git

          if [ -n "$GERRIT_CHANGE_NUMBER" ] ; then
            # Post reviews only for jobs triggered by Gerrit
            automation-git/scripts/jenkins/ardana/gerrit/gerrit_review.py \
              --vote 0 \
              --label 'Verified' \
              --message "
Started build (${JOB_NAME}): ${BUILD_URL}
The following links can also be used to track the results:

- live console output: ${BUILD_URL}console
- live pipeline job view: ${RUN_DISPLAY_URL}
" \
              --patch ${GERRIT_PATCHSET_NUMBER} \
              ${GERRIT_CHANGE_NUMBER}
          fi
        ''')

      }
    }

    stage('validate commit message') {
      when {
        expression { env.GERRIT_CHANGE_COMMIT_MESSAGE != null }
      }
      steps {
        sh '''
          export LC_ALL=C.UTF-8
          export LANG=C.UTF-8

          echo $GERRIT_CHANGE_COMMIT_MESSAGE | base64 --decode | gitlint -C automation-git/scripts/jenkins/gitlint.ini
        '''
      }
    }

    stage('integration test') {
      steps {
        script {
          // reserve a resource here for the openstack-ardana job, to avoid
          // keeping a cloud-ardana-ci worker busy while waiting for a
          // resource to become available.
          // if not instructed to reserve a resource, use a dummy resource and a zero quantity
          // to fool Jenkins into thinking it reserved a resource when in fact it didn't
          lock(label: reserve_env == 'true' ? ardana_env:'dummy-resource',
               variable: 'reserved_env',
               quantity: reserve_env == 'true' ? 1:0 ) {
            if (env.reserved_env && reserved_env != null) {
              env.ardana_env = reserved_env
            }

            def slaveJob = build job: 'openstack-ardana', parameters: [
              string(name: 'ardana_env', value: "$ardana_env"),
              string(name: 'reserve_env', value: "false"),
              string(name: 'cleanup', value: "on success"),
              string(name: 'gerrit_change_ids', value: "$gerrit_change_ids"),
              string(name: 'git_automation_repo', value: "$git_automation_repo"),
              string(name: 'git_automation_branch', value: "$git_automation_branch"),
              string(name: 'scenario_name', value: "standard"),
              string(name: 'clm_model', value: "standalone"),
              string(name: 'controllers', value: "2"),
              string(name: 'sles_computes', value: "1"),
              string(name: 'cloudsource', value: "$cloudsource"),
              string(name: 'ses_enabled', value: "true"),
              string(name: 'ses_rgw_enabled', value: "false"),
              string(name: 'tempest_filter_list', value: "$tempest_filter_list"),
              string(name: 'os_cloud', value: "$os_cloud")
            ], propagate: false, wait: true
            env.jobResult = slaveJob.getResult()
            env.jobUrl = slaveJob.buildVariables.blue_ocean_buildurl
            def jobMsg = "Build ${jobUrl} completed with: ${jobResult}"
            echo jobMsg
            if (env.jobResult != 'SUCCESS') {
               error(jobMsg)
            }
          }
        }
      }
    }
  }
  post {
    always {
      script{
        env.BUILD_RESULT = currentBuild.currentResult
        sh('''
          automation-git/scripts/jenkins/jenkins-job-pipeline-report.py \
            --recursive \
            --filter 'Declarative: Post Actions' \
            --filter 'Setup workspace' > pipeline-report.txt || :

          # Post reviews only for jobs triggered by Gerrit
          if [ -n "$GERRIT_CHANGE_NUMBER" ] ; then
            if [[ $BUILD_RESULT == SUCCESS ]]; then
              vote=+2
              message="
Build succeeded (${JOB_NAME}): ${BUILD_URL}

"
            else
              vote=-2
              message="
Build failed (${JOB_NAME}): ${BUILD_URL}

"
            fi
            automation-git/scripts/jenkins/ardana/gerrit/gerrit_review.py \
              --vote $vote \
              --label 'Verified' \
              --message "$message" \
              --message-file pipeline-report.txt \
              --patch ${GERRIT_PATCHSET_NUMBER} \
              ${GERRIT_CHANGE_NUMBER}

            if [[ $BUILD_RESULT == SUCCESS ]]; then
              automation-git/scripts/jenkins/ardana/gerrit/gerrit_merge.py \
                --patch ${GERRIT_PATCHSET_NUMBER} \
                ${GERRIT_CHANGE_NUMBER}
            fi
          fi
        ''')

      }
    }
    cleanup {
      cleanWs()
    }
  }
}
