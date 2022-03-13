pipeline {
    environment {
        DEPLOY = "${env.BRANCH_NAME == "main" || env.BRANCH_NAME.contains("features") ? "true" : "false" || env.BRANCH_NAME.contains("develop") ? "true" : "false"}"
        HELM_RELEASE = 'notejam'
        REGISTRY = "swlidoc/notejamapp" // Replace with your own repository and image name
        REGISTRY_CREDENTIAL = 'dockerhub-push-pull' // setup this as username/password dockerhub private registry credential in Jenkins
        dockerImage = '' // do not change this
        imagename = "${REGISTRY}:$GIT_COMMIT"
        deployToLocal = true // accepted values : false/true . Set to true to deploy to same cluster where Jenkins instance is running.
        kubeconfig = "REPLACE_ME" // Set to kubeconfig credential ID for deploying to required target, deployToLocal must be set to false
    }
    agent {
        kubernetes {
            defaultContainer 'jnlp'
            yamlFile 'build.yaml'
        }
    }
    stages {
        stage('Docker Build') {
            when {
                environment name: 'DEPLOY', value: 'true'
            }
            steps {
                script {
                    container('ubuntu') {
                        sh "apt update && apt upgrade -y && apt install curl -y && apt install sudo -y"
                        sh "curl -fsSL https://get.docker.com/ | sh"
                        sh "echo \"limit nofile 262144 262144\" >> /etc/init/docker.conf"
                        sh "sudo service docker start"
                        sh "sudo chmod 666 /var/run/docker.sock"
                        sh "sleep 10"
                        sh "docker --version"
                        dockerImage = docker.build REGISTRY + ":$GIT_COMMIT"
                    }
                }
            }
        }
        stage('Docker Publish') {
            when {
                environment name: 'DEPLOY', value: 'true'
            }
            steps {
                script {
                    container('ubuntu') {
                        docker.withRegistry('', REGISTRY_CREDENTIAL) {
                            dockerImage.push()
                        }
                    }
                }
            }
        }
        stage('Kubernetes Deploy') {
            when {
                environment name: 'DEPLOY', value: 'true'
            }
            steps {
                script {
                    container('ubuntu') {
                        sh 'curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl'
                        sh "chmod +x ./kubectl && mv ./kubectl /usr/local/bin"
                        sh "curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3"
                        sh "sudo chmod 700 get_helm.sh"
                        sh "./get_helm.sh"
                        if (env.deployToLocal == 'false') {
                            withCredentials([file(credentialsId: kubeconfig, variable: 'KUBECRED')]) {
                                sh 'mkdir ~/.kube'
                                sh 'cat $KUBECRED > ~/.kube/config'
                                sh "kubectl get nodes"
                            }
                        }
                        withCredentials([
                            usernamePassword(credentialsId: REGISTRY_CREDENTIAL, usernameVariable: 'imageCredentialsUser', passwordVariable: 'imageCredentialsPass')
                        ]){
                            sh ('helm upgrade --install --force --set deployment.image=$imagename --set imageCredentials.username=$imageCredentialsUser --set imageCredentials.password=$imageCredentialsPass $HELM_RELEASE ./notejam')            
                        }
                    }
                }
            }
        }
    }
}