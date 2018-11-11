#! /usr/bin/env groovy

@NonCPS
def getAuthorsEmailsString() {
    def authorsEmails = ""
    print "Changes #[" + currentBuild.changeSets.size() + "]"
    currentBuild.changeSets.each { set ->
        set.each { entry ->
            authorsEmails += entry.author.getProperty(hudson.tasks.Mailer.UserProperty.class).address + ","
        }
    }
    "${(!authorsEmails.trim().isEmpty()) ? authorsEmails[0 .. -1] : 'MDCP-MDM_Contact-Interest@groups.int.hpe.com'}"
}

@NonCPS
def getAuthorsEmails() {
    def authorsEmails = []
    currentBuild.changeSets.each { set ->
        set.each { entry ->
            authorsEmails << entry.author.getProperty(hudson.tasks.Mailer.UserProperty.class).address
        }
    }
    authorsEmails
}

def isPersonalBranch() {
    env.GIT_BRANCH.startsWith('feature/NGITCNT-') || env.GIT_BRANCH.startsWith('enhancement/NGITCNT-')
}

def isPR() {
    env.BRANCH_NAME.startsWith('PR-') && !env.CHANGE_BRANCH.startsWith('PR-')
}

def shouldBranchBuild() {
    isPersonalBranch() ||
        isPR() ||
        env.GIT_BRANCH.startsWith('bugfix/NGITCNT-') ||
        'mainline'.equals(env.GIT_BRANCH) ||
        'master'.equals(env.GIT_BRANCH)
}

def getRepoUser() {
  def matcher = readFile('/var/jenkins_home/.m2/settings.xml') =~ /party.*[\n].*<username>(.+)<\/username>/
  matcher ? matcher[0][1] : null
}

def getRepoPassword() {
  def matcher = readFile('/var/jenkins_home/.m2/settings.xml') =~ /party.*[\n].*[\n].*<password>(.+)<\/password>/
  matcher ? matcher[0][1] : null
}

def readVersion() {
  def matcher = readFile('maven-metadata.xml') =~ '<version>(([0-9]+\\.){2}[0-9]+)-SNAPSHOT.*</version>'
  matcher ? matcher[-1][1] : "1.2.17"
}

def getArtifactFinalNameFromPom() {
  def matcher = readFile('pom.xml') =~ '<artifact.name>(.+)</artifact.name>'
  matcher ? (matcher[0][1] + '.war') : null
}

def newVersion(boolean majorChange, boolean minorChange) {
  def version = readVersion().split('\\.')
  def currMajorVer = version[0]
  def currMinorVer = version[1]
  def currBFEVer= version[2]
  def newMajorVer = majorChange ? (Integer.parseInt(currMajorVer.toString()) + 1).toString() : currMajorVer.toString()
  def newMinorVer = minorChange ? (Integer.parseInt(currMinorVer.toString()) + 1).toString() : currMinorVer.toString()
  newMinorVer = majorChange ? 0 : newMinorVer
  def newBFEVer = majorChange || minorChange ? 0 :  Integer.parseInt(currBFEVer.toString()) + 1
  newMajorVer + '.' + newMinorVer + '.' + newBFEVer
}

def getAppDeploymentServer (String stage) {
  def props = readProperties file: 'src/pipeline/resources/deployment-hosts.properties'
  props.get(stage)
}

@NonCPS
def getStage() {
  def stage = ""
  switch (env.GIT_BRANCH) {
    case 'master':
      stage = 'PROD'
      break
    case 'mainline':
      // If timer build then its an ITG build
      if (null != currentBuild.rawBuild.getCause(hudson.triggers.TimerTrigger$TimerTriggerCause)) {
        stage = 'ITG'
        break
      }
    default:
      stage = 'DEV'
      break
  }
  return stage
}

def doMavenDeployTo(String stage, String serversData) {
  def perServerData = serversData.split(',')
  def addID = perServerData.size() > 1
  for (int i = 0; i < perServerData.size(); i++) {
    def serverAndPort = perServerData[i]
    def host = serverAndPort[0 .. serverAndPort.lastIndexOf(":") - 1]
    def port = serverAndPort[serverAndPort.lastIndexOf(":") + 1 .. - 1]
    def mvnSrvrId="${stage}"
    if (addID) {
      mvnSrvrId += (i + 1)
    }
    print "Deploying to Server='${host}:${port}' using Maven Server ID='${mvnSrvrId}'"
    ///////////////////////////////////////////////////////////////////////////
    ///////////////////////////////////////////////////////////////////////////
    // ALWAYS SKIP TESTS SINCE AT THIS POINT IT WILL BE RUNNING WITH PROD CREDS
    sh "mvn wildfly:redeploy -Pprod,swagger,no-liquibase -DappDplySrvr.id=${mvnSrvrId} -DappDplySrvr.host=${host} -DappDplySrvr.port=${port} -DskipTests"
    ///////////////////////////////////////////////////////////////////////////
    ///////////////////////////////////////////////////////////////////////////
  }
}

def copyConfigurationFiles(String stage, String secretHash){
    sh script: 'rm -f siperian-client.properties'
    sh script: 'rm -f src/main/resources/siperian-client.properties'
    if (stage == 'Prepare PROD Build' ) {
        sh script: '#!/bin/sh -e \n' + "curl -s -O https://raw.github.hpe.com/gist/mdcp-mdm-builder/${secretHash}/raw/siperian-client.properties"
    }else{
        sh script: '#!/bin/sh -e \ncurl -s -O https://raw.github.hpe.com/gist/mdcp-mdm-builder/8050a623d127786ece5ca0bae7a2d33d/raw/siperian-client.properties'
    }
    sh script: 'mv siperian-client.properties src/main/resources/siperian-client.properties'
    sh script: 'rm -f application-prod.yml'
    sh script: 'rm -f src/main/resources/config/application-prod.yml'
    if (stage == 'Prepare PROD Build') {
        sh script: "#!/bin/sh -e \ncurl -s -O https://raw.github.hpe.com/gist/mdcp-mdm-builder/${secretHash}/raw/application-prod.yml"
    }else{
        sh script: '#!/bin/sh -e \ncurl -s -O https://raw.github.hpe.com/gist/mdcp-mdm-builder/8050a623d127786ece5ca0bae7a2d33d/raw/application-prod.yml'
    }
    sh script: 'mv application-prod.yml src/main/resources/config/application-prod.yml'
}

pipeline {
  agent any
  triggers {
    //Runs Thu at 12:00:00 GMT - 6; when is not a date on the `./holiday_calendar`
    cron('H/30 H/18 * * H/4')
  }
  environment {
    TEAM_EMAIL = 'MDCP-MDM_Contact-Interest@groups.int.hpe.com'
    PROJECT_NAME = "${env.GIT_URL[(env.GIT_URL.lastIndexOf('/') + 1) .. -5]}"
    IS_PERSONAL_BRANCH = isPersonalBranch()
    SHOULD_BRANCH_BUILD = shouldBranchBuild()

    // For now initialize WILL_BUILD with the value of SHOULD_BRANCH_BUILD
    WILL_BUILD = "${env.SHOULD_BRANCH_BUILD}"
    IS_PR = isPR()
    AUTHORS_EMAILS_STRING = "${getAuthorsEmailsString()}"
    BASE_MAIL_RECIPIENTS = """${ (env.IS_PERSONAL_BRANCH) 
      ? env.AUTHORS_EMAILS_STRING 
      : env.TEAM_EMAIL }
    """
    AFTER_PR_MAIL_RECIPIENTS = """${ (env.IS_PR)
      ? (env.BASE_MAIL_RECIPIENTS + ',' + env.TEAM_EMAIL)
      : env.BASE_MAIL_RECIPIENTS }
    """

    // Final Mail recipients
    MAIL_RECIPIENTS = "${env.AFTER_PR_MAIL_RECIPIENTS}"
    CURRENT_VERSION = ""
    NEW_VERSION = ""
    CURR_PID = ""

    // Servers
    // Nexus Server Data
    REPO_TO_USE = """${ 'master'.equals(env.GIT_BRANCH)
      ? "maven-releases"
      : "maven-snapshots" }"""
    REPO_ADDR = "hc9t05012.itcs.hpecorp.net:8081"
    REPO_USER =  getRepoUser()
    REPO_PASSWORD = getRepoPassword()
    // App Deploy Server data
    APP_SRVR = "hc9t05013.itcs.hpecorp.net"
    APP_SRVR_URL = "mdm"
    APP_SRVR_DPLY_PATH="/opt/mdm-apps/contact-servies"
    // Maven Site Deploy (Reports/RPRTS) Server data
    RPRTS_SRVR = "hc9t05012.itcs.hpecorp.net"
    RPRTS_SRVR_URL = "mdm"
    SITE_DPLY_PATH="/var/lib/docker/volumes/srv2_nginx/_data/${env.PROJECT_NAME}"

  }
  stages {
    stage('Cleaning and set up') {
      environment {
          STAGE=getStage()
      }
      steps {
        print "Getting latest app version from repo: '${env.GIT_BRANCH}'"
        print "Getting latest app version from repo: '${REPO_TO_USE}'"
        sh script: 'mvn clean generate-sources'
        copyConfigurationFiles("${STAGE}", null)
        sh script: '#!/bin/sh -e \n' + "curl -s -u ${env.REPO_USER}:${env.REPO_PASSWORD} -O http://${REPO_ADDR}/repository/${REPO_TO_USE}/com/hpe/contact/services/contact-services/maven-metadata.xml"
      }
    }
    stage('Copy latest email template'){
      steps {
          print "Getting latest template version"
          sh script: 'mv pipeline-resources/emails/DXC_CI-CD.JOB_STARTING.groovy.email.html /var/jenkins_home/email-templates/DXC_CI-CD.JOB_STARTING.groovy.email.html'
      }
    }
    stage('Send Build Notification') {
      when {
        expression { env.SHOULD_BRANCH_BUILD }
      }
      steps {
        echo 'Notify Build'
        emailext(
            to: "${env.MAIL_RECIPIENTS}",
            /* The line bellow wont work since the Jenkins Git plug in has a bug
            * (https://issues.jenkins-ci.org/browse/JENKINS-9016) that causes the user ID to be taken as the part before
            * the '@' in the e-mail section. This causes it not to match with what is provided by the security realm
            * (currently LDAP) for the ID value, but colliding with the e-mail value which is considered a violation
            * causing an error that looks like:
            *
            * Not sending mail to unregistered user cesar-alonso.martinez-sanchez@hpe.com because your SCM claimed
            * this was associated with a user ID ‘some-example.user-duh' which your security realm does not recognize;
            *	you may need changes in your SCM plugin


            recipientProviders: [[$class: 'CulpritsRecipientProvider']],

            */
            replyTo: "${env.TEAM_EMAIL}",
            mimeType: 'text/html',
            subject: "${env.PROJECT_NAME} Branch ${env.GIT_BRANCH} Build #${env.BUILD_NUMBER} has begun",
            body: '${SCRIPT, template="DXC_CI-CD.JOB_STARTING.groovy.email.html"}');
      }
    }
    stage('Non Functional Testing') {
      parallel {
        stage('Unit Test') {
          steps {
            echo 'Running Unit Test'
            sh 'mvn test -Pprod,swagger,no-liquibase'
          }
        }
      }
    }
    stage('Quality Analysis') {
      parallel {
        stage('SonarQube') {
          steps {
            echo 'Analyzing Code Quality and Vulnerabilities'
            sh 'mvn sonar:sonar -Pprod,swagger,no-liquibase'
          }
        }
      }
    }
    stage('Version Update') {
      steps {
        script {
          CURRENT_VERSION = readVersion()
          print "Current Version = ${CURRENT_VERSION}"
          def userInput = true
          def didTimeout = false
          def timeStamp = Calendar.getInstance().getTime().format('YYYYMMdd-hhmmss',TimeZone.getTimeZone('CST'))
          def waitTime = null
          def timeUnit = null
          if ('mainline'.equals(GIT_BRANCH)) {
            waitTime = 1
            timeUnit = "HOURS"
          }
          if ('master'.equals(GIT_BRANCH)) {
            waitTime = 3
            timeUnit = "HOURS"
          }
          if (waitTime == null || timeUnit == null) {
            waitTime = 6
            timeUnit = "MINUTES"
          }
          try {
                timeout(time: waitTime, unit: timeUnit) {
                  userInput = input (
                    message: 'Please check the boxes that apply for this build to help generate the next proper version value',
                    ok: "Ok this is correct",
                    submitter: "cesar-alonso.martinez-sanchez@hpe.com,zach.noel@hpe.com,marco-antonio.martinez-suarez@hpe.com",
                    parameters: [
                      [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Are there major or backward compatibility breaking changes? This will update the major XX.0.0 part of the version number', name: 'MAJOR_VERSION_CHANGE'],
                      [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Are there new api\'s or significant functionality changes? This will update the minor 0.XX.0 part of the version number', name: 'MINOR_VERSION_CHANGE']
                    ]
                  )
                  NEW_VERSION = newVersion(
                      userInput.get("MAJOR_VERSION_CHANGE"),
                      userInput.get("MINOR_VERSION_CHANGE")
                  )
              }
          } catch(err) {
            def user = err.getCauses()[0].getUser()
            if('SYSTEM' == user.toString()) {
                didTimeout = true
            } else {
                userInput = false
                echo "Aborted by: [${user}]"
            }
            NEW_VERSION = "${CURRENT_VERSION}"
          }
          if (didTimeout) {
            print "No version update was specified. Timpestamping build"
            NEW_VERSION = "${NEW_VERSION}_${timeStamp}"
          }
          if (!'master'.equals("${env.GIT_BRANCH}")) {
            print "This is a Snapshot"
            NEW_VERSION = "${NEW_VERSION}-SNAPSHOT"
          } else {
            print "This is a Release"
          }
        }
        print "New Version = ${NEW_VERSION}"
      }
      post {
        aborted {
          script {
            currentBuild.result = 'SUCCESS'
          }
        }
      }
    }
    stage('Reporting') {
      when {
        expression {
          'mainline'.equals(env.GIT_BRANCH) || 'master'.equals(env.GIT_BRANCH)
        }
      }
      steps {
        print 'Creating site and reports'
        sh '''sed -i '/DEVBLD/d' pom.xml'''
        sh '''sed -i 's/\${appVersion}''' + "/${NEW_VERSION}/g' pom.xml"
        sh '''sed -i 's/appVersion''' + "/${NEW_VERSION}/g' pom.xml"
        sh "mvn clean test site -Pprod,swagger,no-liquibase"
        print 'Publishing Reports'
        sshagent (credentials: ['mdm']) {
          sh "ssh ${RPRTS_SRVR_URL}@${RPRTS_SRVR} 'rm -rf ${SITE_DPLY_PATH}'"
          sh "ssh ${RPRTS_SRVR_URL}@${RPRTS_SRVR} 'mkdir -p ${SITE_DPLY_PATH}'"
          sh "scp -r target/site/*  ${RPRTS_SRVR_URL}@${RPRTS_SRVR}:${SITE_DPLY_PATH}"
        }
      }
    }
    stage('Prepare PROD Build') {
      when {
        branch "master"
      }
      environment {
        SECRET_CONF_HASH=''
        STAGE=getStage()
      }
      steps {
        script {
          echo 'Loading secret variables'
          try {
                timeout(time: 96, unit: "HOURS") {
                  userInput = input (
                    message: 'Please provide secret hash for PROD config',
                    ok: "Try this one",
                    submitter: "cesar-alonso.martinez-sanchez@hpe.com,zach.noel@hpe.com,miguel.santillan@hpe.com,jaime.ruiz@hpe.com",
                    parameters: [
                      [$class: 'TextParameterDefinition', defaultValue: "", description: 'Secret Gist Hash for Environment Configuration', name: 'scu'],
                    ]
                  )
                  SECRET_CONF_HASH = userInput
              }
          } catch(err) {
            def user = err.getCauses()[0].getUser()
            if('SYSTEM' == user.toString()) {
                didTimeout = true
            } else {
                userInput = false
                echo "Aborted by: [${user}]"
            }
          }
        }
        copyConfigurationFiles("${STAGE}", "$SECRET_CONF_HASH")
      }
    }
    stage('Archiving') {
      when {
        expression {
          env.WILL_BUILD
        }
      }
      steps {
        echo 'Packaging application. Ignoring tests since they have already been executed...'
        sh '''sed -i '/DEVBLD/d' pom.xml'''
        sh '''sed -i 's/\${appVersion}''' + "/${NEW_VERSION}/g' pom.xml"
        sh '''sed -i 's/appVersion''' + "/${NEW_VERSION}/g' pom.xml"
        echo 'Nexus Archiving...'
        ///////////////////////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////////////////////
        // ALWAYS SKIP TESTS SINCE AT THIS POINT IT WILL BE RUNNING WITH PROD CREDS
        sh "mvn package deploy:deploy install -DskipTests -Pprod,swagger,no-liquibase"
        ///////////////////////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////////////////////
        script {
          if (!'master'.equals(GIT_BRANCH)) {
            echo 'Local Archiving...'
            archiveArtifacts artifacts: '**/target/*.*ar', fingerprint: true
          }
        }
      }
    }
    stage('Deployment') {
      when {
        expression {
          'mainline'.equals(env.GIT_BRANCH) || 'master'.equals(env.GIT_BRANCH)
        }
      }
      environment {
        DPLYMNT_HOST=''
        DPLYMNT_PORT=''
        STAGE=getStage()
      }
      steps {
        script {
          def serverData = getAppDeploymentServer("${STAGE}")
          DPLYMNT_HOST = serverData[0 .. serverData.lastIndexOf(":") - 1]
          DPLYMNT_PORT = serverData[serverData.lastIndexOf(":") + 1 .. - 1]
          print "Stage = ${STAGE}"
          print "App Deployment Server Address = '${DPLYMNT_HOST}:${DPLYMNT_PORT}'"
          doMavenDeployTo(STAGE, serverData)
        }
        echo "App started"
      }
    }
  }
}
