podTemplate(containers: [
        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'docker', livenessProbe: containerLivenessProbe(execArgs: '', failureThreshold: 0, initialDelaySeconds: 0, periodSeconds: 0, successThreshold: 0, timeoutSeconds: 0), name: 'questcode', resourceLimitCpu: '', resourceLimitMemory: '', resourceRequestCpu: '', resourceRequestMemory: '', ttyEnabled: true, workingDir: '/home/jenkins/agent'),
        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0', livenessProbe: containerLivenessProbe(execArgs: '', failureThreshold: 0, initialDelaySeconds: 0, periodSeconds: 0, successThreshold: 0, timeoutSeconds: 0), name: 'helm-container', resourceLimitCpu: '', resourceLimitMemory: '', resourceRequestCpu: '', resourceRequestMemory: '', ttyEnabled: true)
    ], 
    label: 'questcode', 
    namespace: 'devops', 
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
    ) {
    node('questcode') {
        def REPOS
        def IMAGE_VERSION
        def IMAGE_NAME="frontend"
        def ENVIRONMENT="staging"
        def GIT_REPOS_URL="git@github.com:alexandresampaioo/frontend.git"  
        def CHARTMUSEUM_URL="http://helm-chartmuseum:8080"
        stage('Checkout') {
            echo 'Iniciando clone do repositorio'
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPOS_URL]]])
            IMAGE_VERSION = sh label: '', returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim()
        }
        stage('Package') {
            container('questcode') {
                echo 'Iniciando empacotamento com docker'
                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh label: '', script: "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh label: '', script: "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} . --build-arg NPM_ENV='${ENVIRONMENT}'"
                    sh label: '', script: "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
                }               
            }
        }
        stage('Deploy') {
            container('helm-container'){
                echo 'Iniciando Deploy com helm'
                sh label: '', script: 'helm init --client-only'
                sh label: '', script: "helm repo add questcode ${CHARTMUSEUM_URL}"
                sh label: '', script: 'helm repo update'
                sh label: '', script: "helm upgrade staging-frontend questcode/frontend --set image.tag=${IMAGE_VERSION}"
            }
        }
    }
}
