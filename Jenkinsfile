node {
  def nodeHome
  try {
    notifyBuild('STARTED')

    stage('Preparation') {

      gitCheckThatOut(
        'develop',
        'https://github.com/naz-mul/couch-blog.git')

      nodeHome = tool 'nodejs610'
      env.PATH = "${env.PATH}:${nodeHome}/bin"

      sh 'npm install --global bower'
      sh 'npm install --global grunt-cli'
      sh 'npm install --global karma-cli'

      sh 'npm install'
      sh 'bower install'
    }

    stage('Build') {
      sh 'grunt build'
    }

    stage('Test') {
      sh 'grunt test'
      junit allowEmptyResults: false, testResults: 'reports/test-reports/*.xml'
    }

    stage('Deploy') {
      input message: 'Do you want to deploy to docker?',
        ok: 'PRESS OK TO CONTINUE',
        submitter: 'admin'

      // docker build
      withDockerServer([uri: "unix:///var/run/docker.sock"]) {
        withDockerRegistry([
          credentialsId: '3dbe185d-11aa-419a-ae0b-f85b8a5b5435',
          url: "https://index.docker.io/v1/"]) {

          sh 'docker build -t nalam/couch-blog .'
          sh 'docker push nalam/couch-blog'
          // docker run -p 3000:80 -d nalam/couch-blog
        }
      }

      // save to archive
      archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/**', fingerprint: true, onlyIfSuccessful: true

    }
  } catch (e) {
    // If there was an exception thrown, the build failed
    currentBuild.result = "FAILED"
    throw e
  } finally {
    // Success or failure, always send notifications
    notifyBuild(currentBuild.result)
  }

}


def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus = buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${
    env.BUILD_NUMBER
  }]</a>&QUOT;</p>"""

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  slackSend(color: colorCode, message: summary)

  emailext(
    subject: subject,
    body: details,
    //   recipientProviders: [[$class: 'DevelopersRecipientProvider']],
    attachLog: true,
    mimeType: 'text/html',
    replyTo: 'norelpy@devopsittralee.ie',
    to: 'devopsittralee@gmail.com'
  )
}

/**
 * Clean a Git project workspace.
 * Uses 'git clean' if there is a repository found.
 * Uses Pipeline 'deleteDir()' function if no .git directory is found.
 */
def gitClean() {
  timeout(time: 60, unit: 'SECONDS') {
    if (fileExists('.git')) {
      echo 'Found Git repository: using Git to clean the tree.'
      sh 'git reset --hard'
    } else {
      echo 'No Git repository found: using deleteDir() to wipe clean'
      deleteDir()
    }
  }
}

def gitCheckThatOut(String branch, String vcsUrl) {
  branch = branch ?: 'master'
  // cleanup
  gitClean()
  // checkout
  git branch: "${branch}", url: "${vcsUrl}"
  // get last tag
  sh "git describe --abbrev=0 --tags > .git/tagName"
  tagName = readFile('.git/tagName')
  echo "${tagName}"
  // set DisplayName
  currentBuild.displayName = tagName
}
