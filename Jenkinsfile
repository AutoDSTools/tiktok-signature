properties([
    disableConcurrentBuilds()
])

ECR_REPO = '956449821269.dkr.ecr.us-west-2.amazonaws.com/tiktok-signature'
def ECR_CACHE_REPO = "${ECR_REPO}-cache"
def appList = [
]

def env = ['develop':'staging-scrapers','master':'prod-scrapers']
// BRANCH_NAME may contain the path. Extract the part we need
MAP_OF_BRANCH_NAME = BRANCH_NAME.split('/')
FR_BRANCH_NAME = MAP_OF_BRANCH_NAME[0].replace("_", "-")
TASK_BRANCH_NAME = MAP_OF_BRANCH_NAME[0]
if (MAP_OF_BRANCH_NAME.size() > 1) {
    TASK_BRANCH_NAME = MAP_OF_BRANCH_NAME[1]
}

projectName = JOB_NAME.split('/')[0].toLowerCase()
currentBuild.displayName = "#${currentBuild.id}_${FR_BRANCH_NAME}"
def startDate = new Date()

if (BRANCH_NAME in env.keySet()) {
    NAMESPACE = env[BRANCH_NAME]
} else {
    NAMESPACE = FR_BRANCH_NAME
}

def BUILDX_NAME = "tiktok-signature-${NAMESPACE.toLowerCase()}"
def BUILDX_NAMESPACE = "testing"
def BUILDX_RESOURCE_CPU = "2000m"
def BUILDX_RESOURCE_RAM = "4000Mi"

TAG = "${FR_BRANCH_NAME.toLowerCase()}-${BUILD_NUMBER}"

AGENT = 'master'

if (BRANCH_NAME in env.keySet()) {
    ECR_REPO = "956449821269.dkr.ecr.us-west-2.amazonaws.com/tiktok-signature"
    BRANCH_NAME_JIRA = "${BRANCH_NAME}"
    imageName = "tiktok-signature"
    baseValuesName = valuesName[BRANCH_NAME]
} else {
    ECR_REPO = "956449821269.dkr.ecr.us-west-2.amazonaws.com/tiktok-signature-environment"
    BRANCH_NAME_JIRA = "${TASK_BRANCH_NAME}"
    isEnvironmentBuild = true
    if (BRANCH_NAME.contains('PR-')) {
        isEnvironmentBuild = false
    }
    imageName = "tiktok-signature-environment"
    baseValuesName = "staging"
}

def updateDeploy(appList) {
    appList.each {
        kubectl_command = "${DOCKER_TOOLBOX} kubectl -n ${NAMESPACE} --context=eks set image ${it.value} ${it.key} ${it.key}=${ECR_REPO}:${TAG}"
        sh kubectl_command
        slackSend color: 'good', message: "Successful deployment for ${it.key} in ${NAMESPACE}@EKS"
    }
}

DOCKER_VERSION_TOOLBOX = "public.ecr.aws/autods/devops:alpine-k8s-latest"

DOCKER_BASE_TOOLBOX = "docker run --rm \
    --log-driver none \
    --env-file <(bash -c 'env | grep AWS_') \
    -w /apps \
    -v ~/.kube:/root/.kube \
    -v ~/.aws/:/root/.aws \
    -v ~/.docker/:/root/.docker"

DOCKER_TOOLBOX = "${DOCKER_BASE_TOOLBOX} \
    ${DOCKER_VERSION_TOOLBOX}"

timestamps {
    ansiColor('xterm') {
        node(AGENT) {
            try {

                stage('Git CheckOut') {
                    step([$class: 'WsCleanup'])
                    checkout scm
                }

                stage('Build Docker Image') {
                    withEnv([
                            "NAMESPACE=${NAMESPACE}",
                    ]) {
                        sh """
                            echo "Getting ECR login"
                            ${DOCKER_TOOLBOX} aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 956449821269.dkr.ecr.us-west-2.amazonaws.com
                        """.stripIndent()
                    }
                    def amd64BuildCMD = ""
                        def armBuildCMD = ""
                        parallel(
                            "Build X64": {
                                // Create builder
                                sh """
                                    docker buildx create \
                                        --use \
                                        --append \
                                        --name=${BUILDX_NAME}-amd64 \
                                        --driver=kubernetes \
                                        --node=${BUILDX_NAME}-amd64 \
                                        '--driver-opt=namespace=${BUILDX_NAMESPACE},"nodeselector=environment=General,lifecycle=Ec2Spot,project=Build",requests.cpu=${BUILDX_RESOURCE_CPU},requests.memory=${BUILDX_RESOURCE_RAM},limits.cpu=${BUILDX_RESOURCE_CPU},limits.memory=${BUILDX_RESOURCE_RAM}'
                                """

                                // Make build cmd
                                amd64BuildCMD = "docker buildx build "+
                                    "--builder=${BUILDX_NAME}-amd64 " +
                                    "-f Dockerfile " +
                                    "--cache-to type=registry,ref=${ECR_CACHE_REPO}:${NAMESPACE.toLowerCase()}-amd64,mode=max,image-manifest=true,oci-mediatypes=true " +
                                    "--cache-from type=registry,ref=${ECR_CACHE_REPO}:${NAMESPACE.toLowerCase()}-amd64 " +
                                    "--provenance=false "

                                if (BRANCH_NAME in env.keySet() || isEnvironmentBuild == true) {
                                    amd64BuildCMD +=
                                        "--push " +
                                        "-t ${ECR_REPO}:${TAG}-amd64 "

                                } else {
                                    amd64BuildCMD +=
                                        "--output type=cacheonly "
                                }

                                amd64BuildCMD += "."

                                // Build image with try
                                try {
                                    sh "${amd64BuildCMD}"
                                } catch(error) {
                                    logger("Retry build", "WARNING")
                                    retry(2) {
                                        sh "${amd64BuildCMD}"
                                    }
                                }

                            },
                            "Build ARM": {
                                // Create builder
                                sh """
                                    docker buildx create \
                                        --use \
                                        --append \
                                        --name=${BUILDX_NAME}-arm \
                                        --driver=kubernetes \
                                        --node=${BUILDX_NAME}-arm \
                                        '--driver-opt=namespace=${BUILDX_NAMESPACE},"nodeselector=environment=General,lifecycle=Ec2Spot,project=BuildArm",requests.cpu=${BUILDX_RESOURCE_CPU},requests.memory=${BUILDX_RESOURCE_RAM},limits.cpu=${BUILDX_RESOURCE_CPU},limits.memory=${BUILDX_RESOURCE_RAM}'
                                """

                                // Make build cmd
                                armBuildCMD = "docker buildx build "+
                                    "--builder=${BUILDX_NAME}-arm " +
                                    "-f Dockerfile " +
                                    "--cache-to type=registry,ref=${ECR_CACHE_REPO}:${NAMESPACE.toLowerCase()}-arm,mode=max,image-manifest=true,oci-mediatypes=true " +
                                    "--cache-from type=registry,ref=${ECR_CACHE_REPO}:${NAMESPACE.toLowerCase()}-arm " +
                                    "--provenance=false "

                                if (BRANCH_NAME in env.keySet() || isEnvironmentBuild == true) {
                                    armBuildCMD +=
                                        "--push " +
                                        "-t ${ECR_REPO}:${TAG}-arm "

                                } else {
                                    armBuildCMD +=
                                        "--output type=cacheonly "
                                }

                                armBuildCMD += "."

                                // Build image with try
                                try {
                                    sh "${armBuildCMD}"
                                } catch(error) {
                                    logger("Retry build", "WARNING")
                                    retry(2) {
                                        sh "${armBuildCMD}"
                                    }
                                }
                            }
                        )
                        if (BRANCH_NAME in env.keySet() || isEnvironmentBuild == true) {
                        // Create, push and delete main manifest
                        sh """
                            docker manifest create \
                                ${ECR_REPO}:${TAG} \
                                ${ECR_REPO}:${TAG}-amd64 \
                                ${ECR_REPO}:${TAG}-arm
                            docker manifest push ${ECR_REPO}:${TAG}
                            docker manifest rm ${ECR_REPO}:${TAG}
                        """

                        }
                }

                stage('Check vulnerabilities') {
                    echo "Temporary disabled"
                    // sh """
                    //     docker pull anchore/grype:latest
                    //     docker run --rm \
                    //         -v /var/run/docker.sock:/var/run/docker.sock \
                    //         -v ~/.docker/:/root/.docker \
                    //         anchore/grype:latest \
                    //         -o json \
                    //         docker:${ECR_REPO}:${TAG} \
                    //         | cat > grype-report.json
                    // """
                    // recordIssues(tools: [grype(id: 'grype', name: 'Grype', pattern: '**/grype-report.json')])
                }

                if (BRANCH_NAME in env.keySet()) {
                    stage('Get list of deployments') {
                        withEnv([
                                "NAMESPACE=${NAMESPACE}",
                                "DOCKER_TOOLBOX=${DOCKER_TOOLBOX}"
                        ]) {
                            appListStr = sh (
                                    returnStdout: true,
                                    script: """
                                        ${DOCKER_TOOLBOX} kubectl get -n ${NAMESPACE} --context=eks -o custom-columns=NAME:.metadata.name deployment | grep '^tiktok-signature' | awk '{ quote = "\'\\\'\'"; finish = quote":"quote"deploy"quote; deployStr = quote \$1 finish ; print deployStr}' 
                                    """
                            ).trim()
                        }
                        appList = appListStr.split( '\n' ).collectEntries { entry ->
                            def pair = entry.split(':')
                            [(pair.first()): pair.last()]
                        }
                    }
                    stage("Deploy ${appList}") {
                        updateDeploy(appList)
                    }

                    stage('Check status'){
                        appList.each {
                            kubectl_check = sh(returnStatus: true, script: "${DOCKER_TOOLBOX} kubectl --context=eks rollout status -n ${NAMESPACE} ${it.value}/${it.key} --timeout=20m" )
                            if ( kubectl_check != 0 ){
                                appList.each {
                                    revision = sh(returnStdout: true, script: "${DOCKER_TOOLBOX} kubectl -n ${NAMESPACE} --context=eks rollout history ${it.value}/${it.key}| grep -E '^[0-9]' | tail -n3 | head -n1 | awk {'print\$1'}" )
                                    kubectl_command = "${DOCKER_TOOLBOX} kubectl -n ${NAMESPACE} --context=eks rollout undo ${it.value}/${it.key} --to-revision ${revision}"
                                    sh kubectl_command
                                }
                                currentBuild.result = 'FAILED'
                                slackSend color: 'red', message: "Rollback was performed as ${it.key} failed to deploy"
                                error("Deployment has been crashed on ${it.key} application")
                            }
                        }

                    }
                    def endDate = new Date()
                    def tookTime = groovy.time.TimeCategory.minus(endDate,startDate).toString()
                    slackSend color: 'good', message: "Total deployment time for ${projectName}: ${tookTime}"
                } else {
                    println "Skip deploy for ${BRANCH_NAME} branch"
                }
            } catch (Exception ex) {
                currentBuild.result = 'FAILED'
                throw ex
            } finally {
                // notifySlack(currentBuild.result)

                // Delete builder
                sh """
                    docker buildx rm ${BUILDX_NAME}-amd64
                    docker buildx rm ${BUILDX_NAME}-arm
                """
            }

        }
    }
}


def notifySlack(String buildStatus) {
    buildStatus = buildStatus ?: 'SUCCESS' // build status of null means success

    def endDate = new Date()
    def tookTime = groovy.time.TimeCategory.minus(endDate, startDate).toString()

    def message = "Deployment for `<CHANGE THIS>` in ${NAMESPACE} is ${buildStatus}: #${env.BUILD_NUMBER}:\n Details: ${env.BUILD_URL}\n Total deployment time for orders-api: ${tookTime}"

    def color
    if (buildStatus == 'STARTED') {
        color = '#D4DADF'
    } else if (buildStatus == 'SUCCESS') {
        color = 'good'
    } else if (buildStatus == 'UNSTABLE') {
        color = 'bad'
        message = '@channel ' + message;
    } else {
        color = '#FF9FA1'
        message = '@channel ' + message;
    }
    slackSend(color: color, message: message)
}


// print message to log
// default level is INFO
def logger(String message, String level = 'INFO') {
// Black    0;30
// Blue     0;34
// Green    0;32
// Cyan     0;36
// Red      0;31
// Purple   0;35
// Brown    0;33
// Blue     0;34
// Green    0;32
// Cyan     0;36
// Red      0;31
// Purple   0;35
// Brown    0;33

    def startSequence = '\u001B[0;35m'
    def endSequence = '\u001B[0m';

    if (level == 'INFO') {
        startSequence = '\u001B[0;43m'
    } else if (level == 'WARNING') {
        startSequence = '\u001B[1;31m'
    }

    echo "\n${startSequence}${message}${endSequence}\n"
}
