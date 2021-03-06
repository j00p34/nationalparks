#!/usr/bin/env groovy

//debug output
openshift.verbose(true)
// Application-specific Values
def mavenArgs="-Dcom.redhat.xpaas.repo.redhatga"     // Global maven arguments
def mavenPackageArgs="package spring-boot:repackage" // Maven package arguments
def mavenOutputJar="nationalparks.jar"               // Output jar from Maven
def appName="nationalparks"                          // Application name (for route, buildconfig, deploymentconfig)
def previewAppName="parksmap"                        // Application used for viewing/testing


// Pipeline variables
def isPR=false                   // true if the branch being tested belongs to a PR
def baseProject=""               // name of base project - used when testing a PR
def project=""                   // project where build and deploy will occur
def projectCreated=false         // true if a project was created by this build and needs to be cleaned up
def repoUrl=""                   // the URL of this project's repository

// uniqueName returns a name with a 16-character random character suffix

def generator = { String alphabet, int n ->
  new Random().with { r ->
    (1..n).collect { alphabet[ r.nextInt( alphabet.length() ) ] }.join("")
  }
}


def uniqueName = { String prefix ->
  return prefix + generator( (('a'..'z')+('0'..'9')).join(""), 16 )
}

// setBuildStatus sets a status item on a GitHub commit
def setBuildStatus = { String url, String context, String message, String state, String backref ->
  step([
    $class: "GitHubCommitStatusSetter",
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: url ],
    contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context ],
    errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
    statusBackrefSource: [ $class: "ManuallyEnteredBackrefSource", backref: backref ],
    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [
        [$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

// getRepoURL retrieves the origin URL of the current source repository
def getRepoURL = {
  sh "git config --get remote.origin.url > originurl"
  return readFile("originurl").trim()
}

// getRouteHostname retrieves the host name from the given route in an
// OpenShift namespace
def getRouteHostname = { String routeName, String projectName ->
  sh "oc get route ${routeName} -n ${projectName} -o jsonpath='{ .spec.host }' > apphost"
  return readFile("apphost").trim()
}

// Initialize variables in default node context
node {
  isPR        = env.BRANCH_NAME ? env.BRANCH_NAME.startsWith("PR") : false
  baseProject = env.PROJECT_NAME
  project     = env.PROJECT_NAME
}

try { // Use a try block to perform cleanup in a finally block when the build fails

  node ("maven") {

    stage ('Checkout') {
      checkout scm
      repoUrl = getRepoURL()
      stash includes: "ose3/pipeline-*.json", name: "artifact-template"
    }

    // When testing a PR, create a new project to perform the build 
    // and deploy artifacts.
    openshift.withCluster( 'mycluster' ){
    if (isPR) {
      stage ('Create PR Project') {
        setBuildStatus(repoUrl, "ci/app-preview", "Building application", "PENDING", "")
        setBuildStatus(repoUrl, "ci/approve", "Aprove after testing", "PENDING", "") 
        project = uniqueName("${appName}-")
        echo "Using my-openshift-token for administrative actions"
        openshift.doAs( 'my-openshift-token' ) {
          echo "Create new project for PR using generated name: ${project}"
          openshift.newProject( "${project}" )
          echo "switch to project: ${project}"
          openshift.withProject( "${project}" ) {
              echo "Create jenkins serviceaccount in new project"
              openshift.create('serviceaccount', 'jenkins')
              echo "Add view role to jenkins serviceaccount in new project"
              openshift.policy('add-role-to-user','view','z','jenkins')
              echo "Allow view for all authenicated users"
              openshift.policy('add-role-to-group','view','system:authenticated')
            }
        projectCreated=true
        }
      }
    }
    }
    stage ('Build') {
      sh "mvn clean compile ${mavenArgs}"
    }
    
    stage ('Run Unit Tests') {
      sh "mvn test ${mavenArgs}"
    }

    stage ('Package') {
      sh "mvn ${mavenPackageArgs} ${mavenArgs}"
      sh "mv target/${mavenOutputJar} docker"
      stash includes: "docker/*", name: "dockerbuild"
    //TODO: push built artifact to artifact repository
    }
  }

  node {
    unstash "artifact-template"
    unstash "dockerbuild"

    stage ('Apply object configurations') {
      if (isPR) {
        sh "oc process -f ose3/pipeline-pr-application-template.json -v BASE_NAMESPACE=${baseProject} -n ${project} | oc apply -f - -n ${project}"
      } else {
        sh "oc process -f ose3/pipeline-application-template.json -n ${project} | oc apply -f - -n ${project}"
      }
      // sleep a bit to allow the imagestreams to be populated before we move on to building
      sh "sleep 30"
    }

    stage ('Build Image') {
      sh "oc start-build ${appName}-docker --from-dir=./docker --follow -n ${project}"
    }

    stage ('Deploy') {
      openshiftDeploy deploymentConfig: appName, namespace: project
    }

    if (isPR) {
      stage ('Verify Service') {
        openshiftVerifyService serviceName: previewAppName, namespace: project
      }
      def appHostName = getRouteHostname(previewAppName, project)
      setBuildStatus(repoUrl, "ci/app-preview", "The application is available", "SUCCESS", "http://${appHostName}")
      setBuildStatus(repoUrl, "ci/approve", "Approve after testing", "PENDING", "${env.BUILD_URL}input/") 
      stage ('Manual Test') {
        input "Is everything OK?"
      }
      setBuildStatus(repoUrl, "ci/app-preview", "Application previewed", "SUCCESS", "")
      setBuildStatus(repoUrl, "ci/approve", "Manually approved", "SUCCESS", "")

    }
  }
} 
finally {
  if (projectCreated) {
    node {
      stage('Delete PR Project') {
        sh "oc delete project ${project}"
      }
    }
  }
}
