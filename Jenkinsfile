import groovy.json.JsonSlurper

node {
  stage 'Commit Stage'

    // checkout code from 10.x branch
    git branch: '10.x', url: 'https://github.com/wildfly/quickstart.git'

    // setup JDK
    env.JAVA_HOME="${tool 'JDK_1.8'}"
    env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"

    // setup maven
    def mvnHome = tool name: 'maven_3.3.3', type: 'hudson.tasks.Maven$MavenInstallation'

    // compile and package
    sh "cd kitchensink-angularjs/ && ${mvnHome}/bin/mvn clean install"

    // archive war file
    archive includes: '**/target/*.war'

  stage 'Acceptance tests'
    // download and extract wildfly if not present
    if (!fileExists("${env.JENKINS_HOME}/workspace/${env.JOB_NAME}/wildfly-10.0.0.Final")) {
      sh 'curl http://download.jboss.org/wildfly/10.0.0.Final/wildfly-10.0.0.Final.tar.gz | tar zx'
    }
    env.JBOSS_HOME="${env.JENKINS_HOME}/workspace/${env.JOB_NAME}/wildfly-10.0.0.Final"

    // run acceptance tests
    // the property -Dmaven.test.failure.ignore is used so that the maven command return with exit code 0 on test failures.
    // the test failures are reported as yellow build using the JUnitResultArchiver in the next step
    sh "cd kitchensink-angularjs/ && ${mvnHome}/bin/mvn test -Dmaven.test.failure.ignore -Parq-wildfly-managed"

    // publish results
    step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
}

stage name: 'Manual Tests', concurrency: 1
def parallelStages = [:]
parallelStages["UX Tests"] = {
    node {
      //deploy files
      def warFiles = findFiles glob: '**/target/*.war'
      for (int i=0; i<warFiles.size(); i++) {
        deploy(warFiles[i].path)
      }

      // wait for test feedback
      timeout(time: 5, unit: 'DAYS') {
        input message: 'UX Tests successfull?' /*, submitter: 'qa' */
      }
    }
  }
parallelStages["Security Tests"] = {
    node {
      input message: 'Security Tests successfull?' /*, submitter: 'security' */
    }
  }
parallel parallelStages

stage 'Release'
  node {
    mail (to: 'tobias.getrost@1und1.de',
           from: 'tobias.getrost@getrost.de',
           subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) successfull",
           body: "To download the artifact go to ${env.BUILD_URL}.");
  }

def deploy(deploymentFileName) {
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'wildFlyManagementCredentials', passwordVariable: 'wildflyMgmtPassword', usernameVariable: 'wildflyMgmtUser']]) {
    def hostname = 'localhost'
    def managementPort = '10190'

    def deploymentNameWoPath = determineFileName(deploymentFileName)

    // undeploy old war if present
    sh "curl -S -H \"content-Type: application/json\" -d '{\"operation\":\"undeploy\", \"address\":[{\"deployment\":\"${deploymentNameWoPath}\"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"
    sh "curl -S -H \"content-Type: application/json\" -d '{\"operation\":\"remove\", \"address\":[{\"deployment\":\"${deploymentNameWoPath}\"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"

    echo "Deploying ${deploymentFileName} to ${hostname}:${managementPort} ..."

    // details can be found in: http://blog.arungupta.me/deploy-to-wildfly-using-curl-tech-tip-10/
    // step 1: upload archive
    sh "curl -F \"file=@${deploymentFileName}\" --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management/add-content > result.txt"

    // step 2: deploy the archive
    // read result from step 1
    def uploadResult = readFile 'result.txt'
    def bytesValue = extractByteValue(uploadResult)
    if (bytesValue != null) {
      sh "curl -S -H \"Content-Type: application/json\" -d '{\"content\":[{\"hash\": {\"BYTES_VALUE\" : \"${bytesValue}\"}}], \"address\": [{\"deployment\":\"${deploymentNameWoPath}\"}], \"operation\":\"add\", \"enabled\":\"true\"}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management > result2.txt"
      def deployResult = readFile 'result2.txt'
      def failure = hasFailure(deployResult)
      if (failure != null) {
        error "Deployment of ${deploymentNameWoPath} failed with error: ${failure}"
      }
    } else {
      // fail build as deployment was not successfull
      error "Upload of ${deploymentFileName} failed"
    }
  }
}

@NonCPS
def hasFailure(result) {
  def jsonSlurper = new JsonSlurper()
  def object = jsonSlurper.parseText(result)
  if (object.outcome != 'success') {
    return object.'failure-description'
  } else {
    return null
  }
}

@NonCPS
def extractByteValue(uploadResult) {
  // parse JSON
  def jsonSlurper = new JsonSlurper()
  def object = jsonSlurper.parseText(uploadResult)
  def result = null

  // check that upload was successfull
  if (object.outcome == 'success') {
    result = object.result.BYTES_VALUE
  }
  return result
}

@NonCPS
def determineFileName(path) {
  def idx = path.lastIndexOf('/')
  // Groovy method path.drop(idx) throws a ScriptSecurityException
  return path.substring(idx+1,path.length())
}
