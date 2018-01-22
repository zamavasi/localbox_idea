#!/usr/bin/env groovy

// @TODO investigate better approach than delete dir always in every stage and stashing unstashing between agents. how can we persist the checked-out code? Also no need for custom dirs, Already each stage runs in separate workspace?
// @TODO if we enable concurrent builds per branch we need to make sure code is not corrupt, work with milestones and locks
// @TODO enable version function in shared library step, also update scm commit status,checkout scm and quality gate checker. then create a whole pipeline template
// @TODO ensure gradle tasks dont retrigger tests and builds (for publication) and ensure gradle home dir is available to reuse cached resolved artifacts

/** Pipeline shared variables & credentials **/
def BuildAgentLabel = "testpipe" //@TODO change
def PipelineDockerEphemeralLabel = "${JOB_BASE_NAME}-${BUILD_NUMBER}"
def SlackNotificationChannel = "pipelines"
def SlackTeamDomain = "zamro"
def SlacktokenId = "SLACK_AUTH_TOKEN"
def SonarqubeCredsId = "SONARQUBE_SCANNER_USER"
def SonarQubeUrl = "https://sonar.zamro.com"
def NexusCredsId = "NEXUS3_ZMR_BACKEND_CREDENTIALS"
def GitHubApiKey = "GITHUB_AUTH_TOKEN"
def GitHubSshKey = "GITHUB_SSH_KEY"
def GitServiceUserName = "zamkins"
def GitServiceUserEmail = "zamkins@localhost"
def VersionTrackingFilename = "gradle.properties"

/** helper functions **/

/* add this to release step before starting prompt user and default to patch when {
                expression {
                    boolean publish = false
                    if (env.DEPLOYPAGES == "true") {
                        return true
                    }
                    try {
                        timeout(time: 1, unit: 'MINUTES') {
                            input 'Deploy pages?'
                            publish = true
                        }
                    } catch (final ignore) {
                        publish = false
                    }
                    return publish
                }
            } */

def getProjectVersion(filename) {
    def matcher = readFile(filename) =~ 'version=(.+)'
    matcher ? matcher[0][1].minus("-SNAPSHOT") : null
}

// Use this method if you explicitly create organizationfolders with names equal the actual repo names
def getRepoSlug() {
    tokens = "${env.JOB_NAME}".tokenize('/')
    org = tokens[tokens.size()-3]
    repo = tokens[tokens.size()-2]
    return "${org}/${repo}"
}

// Testing vars //
/* Unit testing */
//NOTE: declare as global var so that can be checked by the pipeline to execute conditionally the test type
unitTestTaskName = "unit"
def unitTestWorkspaceFolder = "unit"

/* Integration testing */
//NOTE: declare as global var so that can be checked by the pipeline to execute conditionally the test type
integrationTestTaskName = "integration"
def integrationTestWorkspaceFolder = "integration"
def integrationDbrootpassword = "zampro"
def integrationDbResourcesVolume = "resources"
def integrationDbHostPort = "32768"
def integrationDbcontainerPort = "3306"
def integrationTestDbConfigurationParams = "mysql --user=root --password=${integrationDbrootpassword} --port=${integrationDbcontainerPort} --protocol=tcp --host=localhost < ${integrationDbResourcesVolume}/create_database_zampro_test.sql"

def getIntegrationDbParams(String IntegrationDbPwd, String DbHostPort, String DbContainerPort) {

    // Docker params
    def integrationDbContainerImage = "mariadb:10.3.2"
    def integrationDbContainerEnvVars = "-e MYSQL_ROOT_PASSWORD=${IntegrationDbPwd}"
    def integrationDbContainerPorts = "-p ${DbHostPort}:${DbContainerPort}"

    def integrationTestContainerParams = "${integrationDbContainerEnvVars} ${integrationDbContainerPorts} ${integrationDbContainerImage}"
    return integrationTestContainerParams
}
def integrationDbContainerName = "mariadb-${PipelineDockerEphemeralLabel}"

// Code quality vars //
def sonarqubeWorkspaceFolder = "sonarqube"

def getSonarQubeParams(String ContainerNameLabel, String WorkspaceRelativePath, String SonarUsername, String SonarPassword, String SonarAnalysisMode, String AnalysisBranch, String ProjVersion, String RepositoryDetails, String GithubKey) {

    // Docker params
    def sonarqubeContainerImageVersion = "1.0.1"
    def sonarqubeContainerImage = "docker.zamro.com/sonar_scanner_alpine:${sonarqubeContainerImageVersion}"
    def sonarqubeContainerName = "--name sonarqube-${ContainerNameLabel}"
    def sonarqubeContainerVolumes = "-v ${WORKSPACE}/${WorkspaceRelativePath}:${WORKSPACE}/${WorkspaceRelativePath}"
    def sonarqubeContainerWorkingDir = "-w ${WORKSPACE}/${WorkspaceRelativePath}"
    def sonarqubeContainerEnvVars = "-e CONTAINER_USER=${SonarUsername}"
    def sonarqubeContainerOverrideCommandEntrypoint = "sonar-scanner -Dsonar.projectBaseDir=."
    def sonarqubeContainerParams = "${sonarqubeContainerName} ${sonarqubeContainerVolumes} ${sonarqubeContainerWorkingDir} ${sonarqubeContainerEnvVars} ${sonarqubeContainerImage} ${sonarqubeContainerOverrideCommandEntrypoint}"

    // Sonarqube Java params
    def sonarqubeServerUrl = "-Dsonar.host.url=https://sonar.zamro.com"
    def sonarqubeCredentialsParams = "-Dsonar.login=${SonarUsername} -Dsonar.password=${SonarPassword}"
    def sonarqubeProjectParams
    if (SonarAnalysisMode == "preview") {
        sonarqubeProjectParams = "-Dsonar.projectVersion=${ProjVersion}" + " " +
                                 "-Dsonar.analysis.mode=preview" + " " +
                                 "-Dsonar.github.pullRequest=${env.CHANGE_ID}" + " " +
                                 "-Dsonar.github.oauth=${GithubKey}" + " " +
                                 "-Dsonar.github.disableInlineComments=false" + " " +
                                 "-Dsonar.github.repository=${RepositoryDetails}"
    }
    else if (SonarAnalysisMode == "fullAnalysis") {
        sonarqubeProjectParams = "-Dsonar.projectVersion=${ProjVersion}"
        if (AnalysisBranch != "master") {
            sonarqubeProjectParams+= " -Dsonar.branch=${AnalysisBranch}"
        }
    }
    else {
        return null
    }

    sonarqubeContainerCommand = "${sonarqubeContainerParams} ${sonarqubeProjectParams} ${sonarqubeServerUrl} ${sonarqubeCredentialsParams}"
    return sonarqubeContainerCommand
}

// Deployment vars //
def releasesWorkspaceFolder = "releases"
def snapshotWorkspaceFolder = "snapshots"

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
                script {
                    if (params.CLEAN_BUILD == true) {
                        echo "Running clean build"
                        deleteDir()
                    }
                }
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
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: "message", state: "SUCCESS"]] ]
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

        stage('Test') {
            // Stop the build if any of the test types fail
            failFast true
            parallel {
                // Assign agents to test steps to take advantage of multiple executors on different nodes
                stage('Unit Tests') {
                    agent {
                        label BuildAgentLabel
                    }
                    when {
                        expression {
                            if (binding.hasVariable('unitTestTaskName') && unitTestTaskName) {
                                return true
                            }
                        }
                    }
                    steps {
                        dir("${unitTestWorkspaceFolder}") {
                            deleteDir()
                            unstash "${JOB_BASE_NAME}"
                            sh "chmod +x ./gradlew && ./gradlew --info ${unitTestTaskName} jacocoGenerateTestReport"
                            publishHTML target: [
                                        allowMissing         : false,
                                        alwaysLinkToLastBuild: false,
                                        keepAll              : true,
                                        reportDir            : "build/reports/jacocoHtml",
                                        reportFiles          : 'index.html',
                                        reportName           : 'Unit tests Coverage report'
                            ]
                            stash name: "${JOB_BASE_NAME}-${unitTestWorkspaceFolder}", includes: "build/jacoco/*.exec"
                        }
                    }
                    post {
                        always {
                            dir("${unitTestWorkspaceFolder}") {
                                publishHTML target: [
                                            allowMissing         : false,
                                            alwaysLinkToLastBuild: false,
                                            keepAll              : true,
                                            reportDir            : "build/reports/HtmlReport/${unitTestTaskName}",
                                            reportFiles          : 'index.html',
                                            reportName           : 'Unit tests report'
                                ]
                                junit allowEmptyResults: false,
                                      testResults: "build/reports/XmlReport/${unitTestTaskName}/TEST-*.xml"
                            }
                            deleteDir()
                        }
                    }
                }
                stage('Integration Tests') {
                    agent {
                        label BuildAgentLabel
                    }
                    when {
                        expression {
                            if (binding.hasVariable('integrationTestTaskName') && integrationTestTaskName) {
                                return true
                            }
                        }
                    }
                    steps {
                        dir("${integrationTestWorkspaceFolder}") {
                            deleteDir()
                            unstash "${JOB_BASE_NAME}"
                            script {
                                integrationDbContainerParams = getIntegrationDbParams(integrationDbrootpassword, integrationDbHostPort, integrationDbcontainerPort)
                            }
                            sh "docker run --name ${integrationDbContainerName} -d ${integrationDbContainerParams}"
                            sh "sleep 15" //@TODO add polling step wait for db is up and running
                            sh "docker exec -i ${integrationDbContainerName} ${integrationTestDbConfigurationParams}"
                            sh "chmod +x ./gradlew && ./gradlew --info ${integrationTestTaskName} jacocoGenerateTestReport"
                            publishHTML target: [
                                        allowMissing         : false,
                                        alwaysLinkToLastBuild: false,
                                        keepAll              : true,
                                        reportDir            : "build/reports/jacocoHtml",
                                        reportFiles          : 'index.html',
                                        reportName           : 'Integration tests Coverage report'
                            ]
                            stash name: "${JOB_BASE_NAME}-${integrationTestWorkspaceFolder}", includes: "build/jacoco/*.exec"
                        }
                    }
                    post {
                        always {
                            echo "Cleaning up ephemeral containers"
                            script {
                                containersToClean = sh (script: "docker ps -aq -f name=${PipelineDockerEphemeralLabel}", returnStdout: true).trim()
                                if (containersToClean) {
                                    sh "docker stop ${containersToClean} && docker rm -v ${containersToClean}"
                                }
                            }
                            dir("${integrationTestWorkspaceFolder}") {
                                publishHTML target: [
                                            allowMissing         : false,
                                            alwaysLinkToLastBuild: false,
                                            keepAll              : true,
                                            reportDir            : "build/reports/HtmlReport/${integrationTestTaskName}",
                                            reportFiles          : 'index.html',
                                            reportName           : 'Integration tests report'
                                ]
                                junit allowEmptyResults    : false,
                                      testResults          : "build/reports/XmlReport/${integrationTestTaskName}/TEST-*.xml"
                            }
                            deleteDir()
                        }
                    }
                }
            }
        }

        stage ('SonarQube Preview PR') {
                    agent {
                        label BuildAgentLabel
                    }
                    when {
                        expression {
                            return env.CHANGE_ID
                        }
                    }
                    environment {
                        //use relevant credentials scoped only to this stage
                        //note if credential type is username/password, the below sets 3 environment variables: <varname> containing <username>:<password>, <varname>_USR containing the username <varname>_PSW containing the password
                        SonarJenkins = credentials("${SonarqubeCredsId}")
                        GithubApiJenkins = credentials("${GitHubApiKey}")
                    }
                    steps {
                        dir("${sonarqubeWorkspaceFolder}") {
                            deleteDir()
                            unstash "${JOB_BASE_NAME}"
                            script {
                                if (binding.hasVariable('unitTestTaskName') && unitTestTaskName) {
                                    unstash "${JOB_BASE_NAME}-${unitTestWorkspaceFolder}"
                                }
                                if (binding.hasVariable('integrationTestTaskName') && integrationTestTaskName) {
                                    unstash "${JOB_BASE_NAME}-${integrationTestWorkspaceFolder}"
                                }
                            }
                            sh "chmod +x ./gradlew && ./gradlew jacocoGenerateCumulativeReport"
                            publishHTML target: [
                                        allowMissing         : false,
                                        alwaysLinkToLastBuild: false,
                                        keepAll              : true,
                                        reportDir            : "build/reports/jacocoCumulativeHtml",
                                        reportFiles          : 'index.html',
                                        reportName           : 'Cumulative tests Coverage report'
                            ]
                            sh "chmod +x ./gradlew && ./gradlew jacocoTestCoverageVerification"
                            script {
                                previewsonar = getSonarQubeParams(PipelineDockerEphemeralLabel, sonarqubeWorkspaceFolder, SonarJenkins_USR, SonarJenkins_PSW, "preview", BRANCH_NAME, CHANGE_ID, repoDetails, GithubApiJenkins_PSW)
                            }
                            sh "docker run -i --tty=false --net=host ${previewsonar}"
                        }
                    }
                    post {
                        always {
                            echo "Cleaning up ephemeral containers"
                            script {
                                containersToClean = sh (script: "docker ps -aq -f name=${PipelineDockerEphemeralLabel}", returnStdout: true).trim()
                                if (containersToClean) {
                                    sh "docker stop ${containersToClean} && docker rm -v ${containersToClean}"
                                }
                            }
                            deleteDir()
                        }
                    }
        }

        stage ('SonarQube Full Analysis') {
                agent {
                    label BuildAgentLabel
                }
                when {
                    not {
                        expression {
                            return env.CHANGE_ID
                        }
                    }
                }
                environment {
                    //use relevant credentials scoped only to this stage
                    SonarJenkins = credentials("${SonarqubeCredsId}")
                }
                steps {
                    dir("${sonarqubeWorkspaceFolder}") {
                        deleteDir()
                        unstash "${JOB_BASE_NAME}"
                        script {
                            if (binding.hasVariable('unitTestTaskName') && unitTestTaskName) {
                                unstash "${JOB_BASE_NAME}-${unitTestWorkspaceFolder}"
                            }
                            if (binding.hasVariable('integrationTestTaskName') && integrationTestTaskName) {
                                unstash "${JOB_BASE_NAME}-${integrationTestWorkspaceFolder}"
                            }
                        }
                        sh "chmod +x ./gradlew && ./gradlew jacocoGenerateCumulativeReport"
                        publishHTML target: [
                                    allowMissing         : false,
                                    alwaysLinkToLastBuild: false,
                                    keepAll              : true,
                                    reportDir            : "build/reports/jacocoCumulativeHtml",
                                    reportFiles          : 'index.html',
                                    reportName           : 'Cumulative tests Coverage report'
                        ]
                        sh "chmod +x ./gradlew && ./gradlew jacocoTestCoverageVerification"
                        script {
                            fullsonar = getSonarQubeParams(PipelineDockerEphemeralLabel, sonarqubeWorkspaceFolder, SonarJenkins_USR, SonarJenkins_PSW, "fullAnalysis", BRANCH_NAME, nextReleaseVersion, "", "")
                        }
                        sh "docker run -i --tty=false --net=host ${fullsonar}"
                        timeout(time: 5, unit: 'MINUTES') {
                            parseQualityGate(SonarQubeUrl,SonarJenkins_USR,SonarJenkins_PSW)
                        }
                    }
                }
                post {
                    always {
                        echo "Cleaning up ephemeral containers"
                        script {
                            containersToClean = sh (script: "docker ps -aq -f name=${PipelineDockerEphemeralLabel}", returnStdout: true).trim()
                            if (containersToClean) {
                                sh "docker stop ${containersToClean} && docker rm -v ${containersToClean}"
                            }
                        }
                        deleteDir()
                    }
                }
        }

        stage('Upload Snapshot Artifact') {
            agent {
                label BuildAgentLabel
            }
            when {
                allOf  {
                    not {
                        expression {
                            return env.CHANGE_ID
                        }
                    }
                    not {
                        branch 'master'
                    }
                }
            }
            steps {
                dir("${snapshotWorkspaceFolder}") {
                    deleteDir()
                    unstash "${JOB_BASE_NAME}"
                    sh "chmod +x ./gradlew && ./gradlew --info ciPublishArchives -Pversion=${BRANCH_NAME}-SNAPSHOT"
                }
            }
            post {
                always {
                    deleteDir()
                }
            }
        }

        stage('Release New Artifact') {
            agent {
                label BuildAgentLabel
            }
            when {
                branch 'master'
            }
            environment {
                //use relevant credentials scoped only to this stage
                Nexus = credentials("${NexusCredsId}")
            }
            steps {
                dir("${releasesWorkspaceFolder}") {
                withCredentials([[
                    $class: 'SSHUserPrivateKeyBinding',
                    credentialsId: "${GitHubSshKey}",
                    usernameVariable: 'GITHUB_USER',
                    keyFileVariable: 'GITHUB_KEY']]) {
                        deleteDir()
                        unstash "${JOB_BASE_NAME}"
                        script {
                            env.GIT_SSH_COMMAND = "ssh -i ${GITHUB_KEY} -l ${GITHUB_USER} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
                        }
                        sh "chmod +x ./gradlew && ./gradlew --info release -Pnexus_user=$Nexus_USR -Pnexus_password=$Nexus_PSW"
                    }
                }
            }
            post {
                always {
                    deleteDir()
                }
            }
        }

        stage('Sanity check prior to Deployment') {
            agent none

            when {
                branch 'master'
            }
            steps{
                timeout(time: 2, unit: 'DAYS') {
                input message: "Do you want to deploy version ${nextReleaseVersion} into production?"
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
