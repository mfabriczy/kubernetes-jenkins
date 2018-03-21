def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
    containerTemplate(name: 'maven-checkstyle', image: 'maven:3.5.2-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven-build', image: 'maven:3.5.2-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'curl', image: 'byrnedo/alpine-curl', ttyEnabled: true, command: 'cat')
  ]) {
    node(label) {
        stage('Checkstyle, build package') {
            checkout scm

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

                        echo 'Deploying package...'
                        kubeCall('k8s/canary/tomcat-canary-deployment.yml', 'https://kubernetes.default.svc.cluster.local/apis/extensions/v1beta1/namespaces/production/deployments')
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

                        echo 'Deploying package...'
                        kubeCall('k8s/dev/tomcat-dev-deployment.yml', 'https://kubernetes.default.svc.cluster.local/apis/extensions/v1beta1/namespaces/' + env.BRANCH_NAME + '/deployments')

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
        -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        -X POST -d "$(cat ' + path + ')" ' + endpoint + ' > /dev/null;'
}
