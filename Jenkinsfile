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
                        echo 'Deploying package...'
                        sh '{ set +x; } 2> /dev/null; \
                            curl -k -s \
                            -H "Content-Type: application/yaml" \
                            -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
                            -X POST -d "$(cat k8s/canary/)" https://kubernetes.default.svc.cluster.local/apis/extensions/v1beta1/namespaces/production/deployments > /dev/null;'
                        break
                }
            }
        }
    }
}
