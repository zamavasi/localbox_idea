#!/usr/bin/env groovy

// Deployment vars //
def releasesWorkspaceFolder = "releases"
def snapshotWorkspaceFolder = "snapshots"

def BuildAgentLabel = "test"

// Shared Library load //
@Library('zamro_shared_library') _

/** Jenkins Pipeline **/
pipeline {
    //no global agent will be allocated for the entire Pipeline run and each stage section will need to contain its own agent. Use checkout step on build time and stash files needed for other stages
    agent none
    // The options directive is for configuration that applies to the whole job.
    options {
        // Make sure we only keep x builds at a time, so we don't fill up our storage!
        buildDiscarder(logRotator(numToKeepStr:'20'))
        // Checkout step explicitly invoked on necessary stage step
        skipDefaultCheckout()
        // "wrapper" steps that should wrap the entire build execution
        timestamps()
        // Global timeout to make sure the pipeline doesn't hang forever
        timeout(time: 3, unit: 'HOURS')
        ansiColor('xterm')
        disableConcurrentBuilds()
  }

    // Access parameters with 'params.PARAM_NAME' - that'll get you default values too.
    parameters {
        booleanParam(name: 'CLEAN_BUILD', defaultValue: false, description: 'Cleanup workspace before building')
    }

    stages {
        stage('Build') {
            agent {
                label BuildAgentLabel
            }
            steps {
                // Customize checkout step in a shared library as the default checkout does not allow custom configuration such as checkout to local branch instead of commit hash
                checkout ([
                    $class: 'GitSCM',
                    branches: scm.branches,
                    extensions: [[$class: 'LocalBranch', localBranch: ""]],
                    userRemoteConfigs: scm.userRemoteConfigs])





script {
  sh "git config --get remote.origin.url > .git/remote-url"
  repoUrl = readFile(".git/remote-url").trim()
  sh "git rev-parse HEAD > .git/current-commit"
  commitSha = readFile(".git/current-commit").trim()
}

echo "${repoUrl}"
echo "${commitSha}"
echo "vasiiii"


step([
    $class: "GitHubCommitStatusSetter",
    contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/checkout"],
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],   //"https://github.com/zarmrocom/testPipeline"], //"git@github.com:zarmrocom/testPipeline"],
    commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
    errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "ERROR"]], // note this needs a check result and throw error else it continues see my tips on scripted error handling
    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "message", state: "ERROR"]] ]
])
sh "sleep 5"
sh "ffff"
            }
            post {
                always {
                    echo "done post" //deleteDir()
                }
            }
        }
        }
    post {
        failure {
                slackSend channel:"${SlackNotificationChannel}",
                          teamDomain: "${SlackTeamDomain}",
                          color: 'danger',
                          message: "${currentBuild.currentResult}: ${env.JOB_NAME} ${env.BUILD_NUMBER}. See ${env.BUILD_URL} for more details",
                          tokenCredentialId: "${SlacktokenId}"
        }
    }
}
