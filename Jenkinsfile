pipeline {
    agent { label 'docker' }

    environment {
        VERSION = sh(returnStdout: true, script: "git tag --contains | head -1").trim()
    }

    stages {
        stage('Build') {
            steps {
                sh "rm -rf pupperware;mv k8s pupperware"
                echo "Helm INIT"
                sh "docker run -i -v ${env.WORKSPACE}:/apps -v pupperware-${VERSION}-${BUILD_NUMBER}:/root/.helm alpine/helm:2.14.1 init --client-only"
                echo "Updating Pupperware Chart Dendencies"
                withCredentials([usernamePassword(credentialsId: 'artifactory', passwordVariable: 'artifactoryPassword', usernameVariable: 'artifactoryUsername')]) 
                {
                    sh "docker run -i -v ${env.WORKSPACE}:/apps -v pupperware-${VERSION}-${BUILD_NUMBER}:/root/.helm alpine/helm:2.14.1 repo add  --username ${artifactoryUsername} --password ${artifactoryPassword} topgolf-helm https://artifactory.topgolf.com/artifactory/topgolf-helm"
                }
                sh "docker run -i -v ${env.WORKSPACE}:/apps -v pupperware-${VERSION}-${BUILD_NUMBER}:/root/.helm alpine/helm:2.14.1 repo add bitnami https://charts.bitnami.com/bitnami"
                sh "docker run -i -v ${env.WORKSPACE}:/apps -v pupperware-${VERSION}-${BUILD_NUMBER}:/root/.helm alpine/helm:2.14.1 dep update /apps/pupperware"
                echo "Building Pupperware Helm Package ${VERSION}"
                sh "docker run -i -v ${env.WORKSPACE}:/apps -v pupperware-${VERSION}-${BUILD_NUMBER}:/root/.helm alpine/helm:2.14.1 package --version '${VERSION}' /apps/pupperware"
            }
        }
        stage('Deploy') {
            steps {
                echo "Publishing Pupperware Helm Cart ${VERSION} to Artifactory"
                withCredentials([usernamePassword(credentialsId: 'artifactory', passwordVariable: 'artifactoryPassword', usernameVariable: 'artifactoryUsername')]) 
                {
                    sh "curl -u${artifactoryUsername}:${artifactoryPassword} -T ${env.WORKSPACE}/pupperware-${VERSION}.tgz https://artifactory.topgolf.com/artifactory/topgolf-helm/pupperware-${VERSION}.tgz"
                    echo "Updating Artifactory Helm Index"
                    sh "curl -u${artifactoryUsername}:${artifactoryPassword} -X POST https://artifactory.topgolf.com/artifactory/api/helm/helm-local/reindex"
                }
                echo "Refreshing Rancher Cataloges"
                withCredentials([string(credentialsId: 'jenkins-rancher-preprod-cli', variable: 'preprodRancherToken')]) 
                {
                    sh "docker run -i -v rancher-cli-preprod-${BUILD_NUMBER}:/root/.rancher rancher/cli2:v2.2.0 login https://rancher-preprod.topgolf.io --token ${preprodRancherToken} --context local:p-6bbpp"
                    sh "docker run -i -v rancher-cli-preprod-${BUILD_NUMBER}:/root/.rancher rancher/cli2:v2.2.0 catalog refresh topgolf-artifactory"
                }
                withCredentials([string(credentialsId: 'jenkins-rancher-prod-cli', variable: 'prodRancherToken')]) 
                {
                    sh "docker run -i -v rancher-cli-prod-${BUILD_NUMBER}:/root/.rancher rancher/cli2:v2.2.0 login https://rancher.prod.topgolf.io --token ${prodRancherToken} --context local:p-zgbbz"
                    sh "docker run -i -v rancher-cli-prod-${BUILD_NUMBER}:/root/.rancher rancher/cli2:v2.2.0 catalog refresh topgolf-artifactory"
                }
            }
        }
    }
}
