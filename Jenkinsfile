def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
  ]) {
    node(label) {
        stage('build') {
            git 'https://github.com/mfabriczy/maven-project.git'
            container('maven') {
                stage('build-maven') {
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
        }
    }
}
