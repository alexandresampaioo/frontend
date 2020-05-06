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
        def IMAGE_POSFIX=""
        def KUBE_NAMESPACE
        def IMAGE_NAME="frontend"
        def ENVIRONMENT
        def GIT_REPOS_URL="git@github.com:alexandresampaioo/frontend.git"  
        def GIT_BRANCH
        def HELM_CHART_NAME="questcode/frontend"
        def HELM_DEPLOY_NAME
        def CHARTMUSEUM_URL="http://helm-chartmuseum:8080"
        def NODE_PORT="30080"

        stage('Checkout') {
            echo 'Iniciando clone do repositorio'
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPOS_URL]]])
            GIT_BRANCH = REPOS.GIT_BRANCH
            if(GIT_BRANCH.equals("origin/master")){
                KUBE_NAMESPACE = "prod"
                ENVIRONMENT="production"
            }else if(GIT_BRANCH.equals("origin/develop")){
                KUBE_NAMESPACE = "staging"
                ENVIRONMENT="staging"
                IMAGE_POSFIX="-RC"
                NODE_PORT="31080"
            }else{
                def error = echo "Nao existe pipeline para a branch ${GIT_BRANCH}"
                echo error
                throw new Exception(error)
            }
            HELM_DEPLOY_NAME=KUBE_NAMESPACE+"-frontend"
            IMAGE_VERSION = sh label: '', returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim()+IMAGE_POSFIX
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
                sh label: '', script: "helm upgrade  --namespace=${KUBE_NAMESPACE} staging-frontend questcode/frontend --set image.tag=${IMAGE_VERSION}"
            
                try{
                    sh label: '', script: "helm upgrade  --namespace=${KUBE_NAMESPACE} ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set service.nodePort=${NODE_PORT} --set image.tag=${IMAGE_VERSION}"
                }catch(Exception e){
                    sh label: '', script: "helm install  --namespace=${KUBE_NAMESPACE} --name ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set service.nodePort=${NODE_PORT} --set image.tag=${IMAGE_VERSION}"
                }
            }
        }
    }
}
