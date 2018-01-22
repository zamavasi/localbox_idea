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
    errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "FAILURE"]], // note this needs a check result and throw error else it continues see my tips on scripted error handling
    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "message", state: "SUdsdsaaCCESS"]] ]
])
sh "sleep 20"
sh "ffff"


withCredentials([[
    $class: 'SSHUserPrivateKeyBinding',
    credentialsId: "${GitHubSshKey}",
    usernameVariable: 'GITHUB_USER',
    keyFileVariable: 'GITHUB_KEY']]) {
        script {
            env.GIT_SSH_COMMAND = "ssh -i ${GITHUB_KEY} -l ${GITHUB_USER} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
        }



  step([
      $class: "GitHubCommitStatusSetter",
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "vasii"],
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/zarmrocom/testPipeline"], //"git@github.com:zarmrocom/testPipeline"],
      commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "FAILURE"]], // note this needs a check result and throw error else it continues see my tips on scripted error handling
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "message", state: "SUCCaaESS"]] ]
  ])
//  error "Pipeline aborted due to quality gate failure"
sh "sleep 20"
step([
    $class: "GitHubCommitStatusSetter",
   contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "nextone!"],
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
  //  commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
    errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "wait what???", state: "PENDING"]] ]
])

sh "ffff"

    }



                //setBuildStatus ("${context}", 'Checking Sonarqube quality gate', 'PENDING')
                // TODO add this to run only when PR for master branch target
                inputVersionBumper()
                // Needed due to JENKINS-43563 - Git plugin does not set user in pipeline
                sh "git config user.name ${GitServiceUserName}"
                sh "git config user.email ${GitServiceUserEmail}"

                echo "Building step for branch: ${JOB_BASE_NAME} #${BUILD_NUMBER}"
                // Necessary to update execute permissions to binaries due to JENKINS-26659
                sh 'chmod +x ./gradlew && ./gradlew --info clean assemble'
                stash name: "${JOB_BASE_NAME}", useDefaultExcludes:false
                script {
                    repoDetails = getRepoSlug()
                    nextReleaseVersion = getProjectVersion(VersionTrackingFilename)
                }
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
