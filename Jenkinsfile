def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
    containerTemplate(name: 'maven-checkstyle', image: 'maven:3.5.2-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven-build', image: 'maven:3.5.2-jdk-8-alpine', ttyEnabled: true, command: 'cat')
  ]) {
    node(label) {
        stage('Checkstyle, build package') {
            git 'https://github.com/mfabriczy/maven-project.git'
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
    }
}
