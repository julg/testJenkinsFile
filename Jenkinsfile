//this is the scripted method with groovy engine
import hudson.model.Result

node {

    //Configuration
    def gitURL = "https://github.com/julg/testJenkinsFile.git"
    def gitCredentials = ''
    def jarDir = "target/"
    def jarName = "AccesCbs"
    def slackChannel = "#notif-api-communes"

    // Variables globales
    def maventool
    def rtMaven
    def buildInfo
    def server
    def serverHostnames = []
    def executeTests

    // Configuration du job Jenkins
    // On garde les 5 derniers builds par branche
    // On scanne les branches et les tags du Git
    properties([
            buildDiscarder(
                    logRotator(
                            artifactDaysToKeepStr: '',
                            artifactNumToKeepStr: '',
                            daysToKeepStr: '',
                            numToKeepStr: '5')
            ),
            parameters([
                    gitParameter(
                            branch: '',
                            branchFilter: 'origin/(.*)',
                            defaultValue: 'main',
                            description: 'Sélectionner la branche ou le tag à déployer',
                            name: 'BRANCH_TAG',
                            quickFilterEnabled: false,
                            selectedValue: 'NONE',
                            sortMode: 'DESCENDING_SMART',
                            tagFilter: '*',
                            type: 'PT_BRANCH_TAG'),
                    booleanParam(defaultValue: false, description: 'Voulez-vous exécuter les tests ?', name: 'executeTests')
            ])
    ])

    stage('Set environnement variables') {
        try {
            env.JAVA_HOME = "${tool 'Open JDK 11'}"
            env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"

            maventool = tool 'Maven 3.3.9'
            rtMaven = Artifactory.newMavenBuild()
            server = Artifactory.server '-1137809952@1458918089773'
            rtMaven.tool = 'Maven 3.3.9'
            rtMaven.opts = '-Xms1024m -Xmx4096m'

            if (params.BRANCH_TAG == null) {
                throw new Exception("Variable BRANCH_TAG is null")
            } else {
                echo "Branch to deploy =  ${params.BRANCH_TAG}"
            }

            if (params.executeTests == null) {
                executeTests = false
            } else {
                executeTests = params.executeTests
            }

            echo "executeTests =  ${executeTests}"

        } catch (e) {
            currentBuild.result = hudson.model.Result.NOT_BUILT.toString()
            notifySlack(slackChannel,e.getLocalizedMessage())
            throw e
        }
    }

    stage('SCM checkout') {
        try {
            checkout([
                    $class                           : 'GitSCM',
                    branches                         : [[name: "${params.BRANCH_TAG}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions                       : [],
                    submoduleCfg                     : [],
                    userRemoteConfigs                : [[credentialsId: "${gitCredentials}", url: "${gitURL}"]]
            ])

        } catch (e) {
            currentBuild.result = hudson.model.Result.FAILURE.toString()
            notifySlack(slackChannel,e.getLocalizedMessage())
            throw e
        }
    }



    stage("Edit properties files") {
      try {
        echo "Edition CommandesTest.properties"
        echo "--------------------------"
        original = readFile "src/test/resources/CommandesTest.properties"
        newconfig = original

       
        withCredentials([
          string(credentialsId: "AccesCbs-connect-login", variable: 'connectLogin')
        ]) {
          newconfig = newconfig.replaceAll("connect.login=*", "connect.login=${connectLogin}")
        }
        withCredentials([
          string(credentialsId: "AccesCbs-connect-password", variable: 'password')
        ]) {
          newconfig = newconfig.replaceAll("connect.password=*", "connect.password=${password}")
        }
        withCredentials([
          string(credentialsId: "AccesCbs-connect-ip", variable: 'ip')
        ]) {
          newconfig = newconfig.replaceAll("connect.ip=*", "connect.ip=${ip}")
        }
        withCredentials([
          string(credentialsId: "AccesCbs-connect-port", variable: 'port')
        ]) {
          newconfig = newconfig.replaceAll("connect.port=*", "connect.port=${port}")
        }

        writeFile file: "src/test/resources/CommandesTest.properties", text: "${newconfig}"

      } catch (e) {
        currentBuild.result = hudson.model.Result.FAILURE.toString()
        notifySlack(slackChannel, "Failed to edit properties files: " + e.getLocalizedMessage())
        throw e
      }
    }


    if ("${executeTests}" == 'true') {
        stage('test') {
            try {

                rtMaven.run pom: 'pom.xml', goals: 'clean test'
                junit allowEmptyResults: true, testResults: '/target/surefire-reports/*.xml'

            } catch (e) {
                currentBuild.result = hudson.model.Result.UNSTABLE.toString()
                notifySlack(slackChannel,e.getLocalizedMessage())
                // Si les tests ne passent pas, on mets le build en UNSTABLE et on continue
                //throw e
            }
        }
    } else {
        echo "Tests are skipped"
    }
	
	
    stage('compile-package') {
        try {
            echo "Compilation..."
            echo "--------------------------"

            sh "'${maventool}/bin/mvn' -Dmaven.test.skip=true clean package -DfinalName='${jarName}'"
        } catch(e) {
            currentBuild.result = hudson.model.Result.FAILURE.toString()
            notifySlack(slackChannel,e.getLocalizedMessage())
            throw e
        }
    }

    stage('sonarqube analysis'){
       withSonarQubeEnv('SonarQube Server'){
          sh "${maventool}/bin/mvn sonar:sonar"
      }
     }

    stage('artifact') {
        try {
            archive "${jarDir}${jarName}.jar"

        } catch(e) {
            currentBuild.result = hudson.model.Result.FAILURE.toString()
            notifySlack(slackChannel,e.getLocalizedMessage())
            throw e
        }
    }

    stage ('Artifactory configuration') {
        try {
            rtMaven.deployer server: server, releaseRepo: 'libs-release-local', snapshotRepo: 'libs-snapshot-local'
            buildInfo = Artifactory.newBuildInfo()
            buildInfo = rtMaven.run pom: 'pom.xml', goals: '-U clean install -Dmaven.test.skip=true '

            rtMaven.deployer.deployArtifacts buildInfo
            buildInfo = rtMaven.run pom: 'pom.xml', goals: 'clean install -Dmaven.repo.local=.m2 -Dmaven.test.skip=true'
            buildInfo.env.capture = true
            server.publishBuildInfo buildInfo

        } catch(e) {
            currentBuild.result = hudson.model.Result.FAILURE.toString()
            notifySlack(slackChannel,e.getLocalizedMessage())
            throw e
        }
    }

    currentBuild.result = hudson.model.Result.SUCCESS.toString()
    notifySlack(slackChannel,"Congratulation !")
}

def notifySlack(String slackChannel, String info = '') {
    def colorCode = '#848484' // Gray

    switch (currentBuild.result) {
        case 'NOT_BUILT':
            colorCode = '#FFA500' // Orange
            break
        case 'SUCCESS':
            colorCode = '#00FF00' // Green
            break
        case 'UNSTABLE':
            colorCode = '#FFFF00' // Yellow
            break
        case 'FAILURE':
            colorCode = '#FF0000' // Red
            break;
    }

    String message = """
        *Jenkins Build*
        Job name: `${env.JOB_NAME}`
        Build number: `#${env.BUILD_NUMBER}`
        Build status: `${currentBuild.result}`
        Branch or tag: `${params.BRANCH_TAG}`
        Target environment: `${params.ENV}`
        Message: `${info}`
        Build details: <${env.BUILD_URL}/console|See in web console>
    """.stripIndent()

    return slackSend(tokenCredentialId: "slack_token",
            channel: "${slackChannel}",
            color: colorCode,
            message: message)
}
