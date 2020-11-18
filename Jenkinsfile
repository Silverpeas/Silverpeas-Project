import java.util.regex.Matcher

node {
  catchError {
    def version
    docker.image('silverpeas/silverbuild')
        .inside('-v $HOME/.m2:/home/silverbuild/.m2 -v $HOME/.gitconfig:/home/silverbuild/.gitconfig -v $HOME/.ssh:/home/silverbuild/.ssh -v $HOME/.gnupg:/home/silverbuild/.gnupg') {
      stage('Preparation') {
        checkout scm
        version = computeSnapshotVersion()
        sh """
sed -i -e "s/<silverpeas.bom.version>[0-9a-zA-Z.-]\\+/<silverpeas.bom.version>${version}/g" pom.xml
mvn -U versions:set -DgenerateBackupPoms=false -DnewVersion=${version}
"""
      }
      stage('Build') {
        waitForDependencyRunningBuildIfAny(version, 'testdep')
        waitForDependencyRunningBuildIfAny(version, 'dep')
        def lockFilePath = createLockFile(version, 'project')
        def status = sh (returnStatus: true,
            script: "mvn clean install -Djava.awt.headless=true")
        deleteLockFile(lockFilePath)
        if (status != 0) {
          error "Build Failure"
        }
      }
    }
  }
  step([$class                  : 'Mailer',
        notifyEveryUnstableBuild: true,
        recipients              : "miguel.moquillon@silverpeas.org, yohann.chastagnier@silverpeas.org, nicolas.eysseric@silverpeas.org",
        sendToIndividuals       : true])
}

def computeSnapshotVersion() {
  def pom = readMavenPom()
  final String version = pom.version
  final String release = pom.properties['next.release']
  final String defaultVersion = env.BRANCH_NAME == 'master' || env.BRANCH_NAME.endsWith('.x') ?
      version : release + '-' + env.BRANCH_NAME.toLowerCase().replaceAll('[# -]', '')
  Matcher m = env.CHANGE_TITLE =~ /^(Bug #?\d+|Feature #?\d+).*$/
  String snapshot = m.matches()
      ? m.group(1).toLowerCase().replaceAll(' #?', '')
      : ''
  if (snapshot.isEmpty()) {
    m = env.CHANGE_TITLE =~ /^\[([^\[\]]+)].*$/
    snapshot = m.matches()
        ? m.group(1).toLowerCase().replaceAll('[/><|:&?!;,*%$=}{#~\'"\\\\°)(\\[\\]]', '').trim().replaceAll('[ @]', '-')
        : ''
  }
  return snapshot.isEmpty() ? defaultVersion : "${release}-${snapshot}"
}

static def createLockFilePath(version, projectName) {
  final String lockFilePath = "\$HOME/.m2/${version}_${projectName}_build.lock"
  return lockFilePath
}

def createLockFile(version, projectName) {
  final String lockFilePath = createLockFilePath(version, projectName)
  sh "touch ${lockFilePath}"
  return lockFilePath
}

def deleteLockFile(lockFilePath) {
  if (isLockFileExisting(lockFilePath)) {
    sh "rm -f ${lockFilePath}"
  }
}

def isLockFileExisting(lockFilePath) {
  if (lockFilePath?.trim()?.length() > 0) {
    def exitCode = sh script: "test -e ${lockFilePath}", returnStatus: true
    return exitCode == 0
  }
  return false
}

def waitForDependencyRunningBuildIfAny(version, projectName) {
  final String dependencyLockFilePath = createLockFilePath(version, projectName)
  timeout(time: 3, unit: 'HOURS') {
    waitUntil {
      return !isLockFileExisting(dependencyLockFilePath)
    }
  }
  if (isLockFileExisting(dependencyLockFilePath)) {
    error "After timeout dependency lock file ${dependencyLockFilePath} is yet existing!!!!"
  }
}
