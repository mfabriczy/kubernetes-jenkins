def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
      containerTemplate(name: 'maven-checkstyle', image: 'maven:3.6.1-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
      containerTemplate(name: 'maven-build', image: 'maven:3.6.1-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
      containerTemplate(name: 'curl', image: 'byrnedo/alpine-curl', ttyEnabled: true, command: 'cat'),
      containerTemplate(name: 'docker', image: 'docker:20.10.1', ttyEnabled: true, command: 'cat')],
      volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]) {
    node(label) {
        checkout scm

        def username = 'mfabriczy'
        def appName = 'tomcat-war-deploy'
        def imageTag = "${username}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

        stage('Checkstyle, build package') {
            parallel (
                checkstyle: {
                    container('maven-checkstyle') {
                        sh 'mvn checkstyle:checkstyle'
                    }
                },
                build: {
                    container('maven-build') {
                        try {
                            sh 'mvn clean package'

                            echo 'Now Archiving...'
                            archiveArtifacts artifacts: '**/target/*.war'
                        } catch (ex) {
                            echo 'Unable to build Maven package.'
                            throw ex
                        }
                    }
                }
            )
        }

        stage('Build and push Docker image') {
            withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                container('docker') {
                    sh "docker build -t ${imageTag} ."
                    sh 'docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_PASSWORD'
                    sh "docker push ${imageTag}"
                }
            }
        }

        stage ('Deploy package') {
            container('curl') {
                switch (env.BRANCH_NAME) {
                    case "canary":
                        echo '''
                        -------------------------------
                        Updating canary environment...
                        -------------------------------
                        '''
                        echo 'Deploying service...'
                        kubeCall('k8s/services/tomcat-service.yml', 'https://kubernetes.default.svc.cluster.local/api/v1/namespaces/production/services')

                        sh "sed -i 's#mfabriczy/tomcat-war-deploy:1.0#${imageTag}#' k8s/canary/tomcat-canary-deployment.yml"

                        echo 'Deploying package...'
                        kubeCall('k8s/canary/tomcat-canary-deployment.yml', 'https://kubernetes.default.svc.cluster.local/apis/apps/v1/namespaces/production/deployments')
                        break
                    case "master":
                        echo '''
                        ----------------------------------
                        Updating production environment...
                        ----------------------------------
                        '''
                        echo 'Deploying service...'
                        kubeCall('k8s/services/tomcat-service.yml', 'https://kubernetes.default.svc.cluster.local/api/v1/namespaces/production/services')

                        sh "sed -i 's#mfabriczy/tomcat-war-deploy:1.0#${imageTag}#' k8s/production/tomcat-prod-deployment.yml"

                        echo 'Deploying package...'
                        kubeCall('k8s/production/tomcat-prod-deployment.yml', 'https://kubernetes.default.svc.cluster.local/apis/apps/v1/namespaces/production/deployments')
                        break
                    default:
                        echo '''
                        ------------------------------------
                        Deploying development environment...
                        ------------------------------------
                        '''
                        echo "Creating namespace: ${env.BRANCH_NAME}"
                        sh "sed -i 's/dev/${env.BRANCH_NAME}/' k8s/dev/namespace/dev-namespace.yml"
                        kubeCall('k8s/dev/namespace/dev-namespace.yml', 'https://kubernetes.default.svc.cluster.local/api/v1/namespaces')

                        echo 'Deploying service...'
                        sh "sed -i 's/LoadBalancer/ClusterIP/' k8s/services/tomcat-service.yml"
                        kubeCall('k8s/services/tomcat-service.yml', 'https://kubernetes.default.svc.cluster.local/api/v1/namespaces/' + env.BRANCH_NAME + '/services')

                        sh "sed -i 's#mfabriczy/tomcat-war-deploy:1.0#${imageTag}#' k8s/dev/tomcat-dev-deployment.yml"

                        echo 'Deploying package...'
                        kubeCall('k8s/dev/tomcat-dev-deployment.yml', 'https://kubernetes.default.svc.cluster.local/apis/apps/v1/namespaces/' + env.BRANCH_NAME + '/deployments')

                        echo "Run `kubectl proxy`. You can access the environment via http://localhost:<port>/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/tomcat-service:8080/"
                }
            }
        }
    }
}

def kubeCall(path, endpoint) {
    sh '{ set +x; } 2> /dev/null; \
        curl -k -s \
        -H "Content-Type: application/yaml" \
        -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" \
        -X POST -d "$(cat ' + path + ')" ' + endpoint + ' > /dev/null;'
}
