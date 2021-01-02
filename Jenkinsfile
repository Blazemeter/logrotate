clearWorkspaceAsRoot()

@Library ('jenkins_library') _
import com.blazemeter.pr.PullRequestUtils
import com.blazemeter.pr.PullRequestStatus
import com.blazemeter.jenkins.lib.DockerTag

properties([pipelineTriggers([githubPush()])]) //Enable Git webhook triggering
pipeline
{
    agent {
        docker {
            image 'us.gcr.io/verdant-bulwark-278/jenkins-docker-agent:master.82'
            args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    parameters {
        booleanParam(name: 'PERFORM_PRISMA_SCAN', defaultValue: true, description: 'Perform a Prisma scan for the local image')
        booleanParam(name: 'PERFORM_PRISMA_SCAN', defaultValue: false, description: 'Perform a Prisma scan for the local image')
        booleanParam(name: 'FAIL_JOB_ON_SCAN_FAILURES', defaultValue: false, description: 'If checked, Twistlock vulnerabilities scan will enforce job failure.', )
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '1000', daysToKeepStr: '180'))
        ansiColor('xterm')
        timeout time: 30, unit: 'MINUTES'
        timestamps()
        disableConcurrentBuilds()
    }
    stages {
        stage('Init') {
            steps {
                script {
                    PullRequestUtils.updateBranchPullRequestsStatuses(this, PullRequestStatus.PENDING)
                    hostname = 'us.gcr.io'
                    project = 'verdant-bulwark-278'
                    appName = 'logrotate'
                    fullImageName = "${hostname}/${project}/${appName}"
                    imageBaseName = "${appName}:${env.BRANCH_NAME}"
                    imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    imageTagLatest = "${imageBaseName}.latest"
                    dockerTag = new DockerTag()
                    dockerTag.allTags << imageTag
                    dockerTag.allTags << "${env.BRANCH_NAME}.latest"

                    commitDate = sh script: 'git log -1 --format="%ad %H"', returnStdout: true
                    sh """
                        echo -n '$JOB_NAME $BUILD_NUMBER $env.BRANCH_NAME $env.GIT_COMMIT $commitDate' > VERSION
                       """
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    docker.build(imageTagLatest, ' -f Dockerfile .')
                }
            }
        }
        stage('Deploy to GCR') {
            when { expression { params.TEST_JENKINS_BUILD == false  } }
            steps {
                script {
                    pushImageToAllRegistries(appName, imageTagLatest, dockerTag)
                }
            }
        }
        stage('Perform PrismaCloud scan') {
            when {  allOf {
                    expression {    params.PERFORM_PRISMA_SCAN ==  true }
                    expression {   params.TEST_JENKINS_BUILD ==  false }
            }
        }
            steps {
                script {
                    runPrismaCloudScan(fullImageName, imageTag, params.FAIL_JOB_ON_SCAN_FAILURES)
                }
            }
    }
}
    post {
        always {
            smartSlackNotification(alternateJobTitle: 'blazemeter logrotate package build')
            script {
                PullRequestUtils.updateBranchPullRequestsStatuses(this)
            }
        }
    }
}
