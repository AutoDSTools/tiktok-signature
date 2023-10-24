ECR_REPO = '956449821269.dkr.ecr.us-west-2.amazonaws.com/tiktok-signature'
def appList = [
]

def env = ['develop':'staging-scrapers','master':'prod-scrapers']
projectName = JOB_NAME.split('/')[0].toLowerCase()
TAG = "${BRANCH_NAME}-0.1.${BUILD_NUMBER}"
currentBuild.displayName = "#${currentBuild.id}_${TAG}"
def startDate = new Date()

if (BRANCH_NAME in env.keySet()) {
    NAMESPACE = env[BRANCH_NAME]
} else {
    NAMESPACE = 'staging'
}
if (BRANCH_NAME.contains('PR-')) {
    AGENT = 'jenkins-slave'
} else {
    AGENT = 'master'
}

def updateDeploy(appList) {
    appList.each {
        kubectl_command = "${DOCKER_TOOLBOX} kubectl -n ${NAMESPACE} --context=eks set image ${it.value} ${it.key} ${it.key}=${ECR_REPO}:${TAG}"
        sh kubectl_command
        slackSend color: 'good', message: "Successful deployment for ${it.key} in ${NAMESPACE}@EKS"
    }
}

DOCKER_VERSION_TOOLBOX = "public.ecr.aws/b0y9w6l0/devops:alpine-k8s-1.24.10"

DOCKER_BASE_TOOLBOX = "docker run --rm \
    --log-driver none \
    --env-file <(bash -c 'env | grep AWS_') \
    -v ~/.kube:/root/.kube \
    -v ~/.aws/:/root/.aws \
    -v ~/.docker/:/root/.docker"

DOCKER_TOOLBOX = "${DOCKER_BASE_TOOLBOX} \
    ${DOCKER_VERSION_TOOLBOX}"

DOCKER_TOOLBOX_HELM = "${DOCKER_BASE_TOOLBOX} \
    -w /apps \
    -v \$(pwd):/apps \
    ${DOCKER_VERSION_TOOLBOX}"

timestamps {
    ansiColor('xterm') {
        node(AGENT) {

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
                docker_image = docker.build("${ECR_REPO}:${TAG}",'--no-cache -f Dockerfile .')
                if (BRANCH_NAME in env.keySet()) {
                    docker_image.push()
                }
            }

            if (BRANCH_NAME in env.keySet()) {
                stage('Get list of deployments') {
                    withEnv([
                            "NAMESPACE=${NAMESPACE}",
                            "DOCKER_TOOLBOX=${DOCKER_TOOLBOX}"
                    ]) {
                        appListStr = sh (
                                returnStdout: true,
                                script: '${DOCKER_TOOLBOX} kubectl get -n ${NAMESPACE} --context=eks -o custom-columns=NAME:.metadata.name deployment | grep \'^tiktok-signature\' | awk \'{ quote = "\'\\\'\'"; finish = quote":"quote"deploy"quote; deployStr = quote $1 finish ; print deployStr}\' '
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

        }
    }
}
