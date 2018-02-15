#!groovy

// Run this node on a Maven Slave
// Maven Slaves have JDK and Maven already installed
node('maven') {
  // Make sure your nexus_openshift_settings.xml
  // Is pointing to your nexus instance
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  stage('Checkout Source') {
    // Get Source Code from SCM (Git) as configured in the Jenkins Project
    // Next line for inline script, "checkout scm" for Jenkinsfile from Gogs
    //git 'http://gogs-demo-mus-cicd.apps.2245.openshift.opentlc.com/mmusaji/kitchensink.git'
    checkout scm
    sh("git config --global user.email 'mmusaji@redhat.com'")
    sh("git config --global user.name 'Mustafa Musaji'")
    sh("git tag -a 7.0.'${currentBuild.number}' -m 'Jenkins'")
    sh('git push http://mmusaji:redhat@gogs-demo-mus-cicd.apps.2245.openshift.opentlc.com/mmusaji/kitchensink.git --tags')
  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = "7.0.${currentBuild.number}"

  stage('Build war') {
    echo "Building version ${version}"

    sh "${mvnCmd} versions:set -DnewVersion=7.0.${version}"
    sh "${mvnCmd} clean package -DskipTests -Popenshift"
  }
  stage('Unit Tests') {
    echo "Unit Tests"
  //  sh "${mvnCmd} test"
  }
  stage('Code Analysis') {
    echo "Code Analysis"

    // Replace xyz-sonarqube with the name of your project
    //sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-mus-cicd.apps.2245.openshift.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME}"
  }

  stage('Publish to Nexus') {
    echo "Publish to Nexus"

    // Replace xyz-nexus with the name of your project
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.mus-cicd.svc.cluster.local:8081/repository/releases"
  }

  stage('Build OpenShift Image') {
    def newTag = "TestingCandidate-${version}"
    echo "New Tag: ${newTag}"

    // Copy the war file we just built and rename to ROOT.war
    sh "cp ./target/jboss-kitchensink-angularjs.war ./ROOT.war"

    // Start Binary Build in OpenShift using the file we just published
    // Replace xyz-tasks-dev with the name of your dev project
    sh "oc whoami"
    sh "oc project mus-kitchensink-dev"
    sh "oc start-build kitchensink --follow --from-file=./deployments/ROOT.war -n mus-kitchensink-dev"

    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  stage('Deploy to Dev') {
    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
    // Replace xyz-tasks-dev with the name of your dev project
    sh "oc project mus-kitchensink-dev"
    sh "oc patch dc kitchensink --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"kitchensink\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:TestingCandidate-$version\"}}}]}}' -n mus-kitchensink-dev"

    openshiftDeploy depCfg: 'kitchensink', namespace: 'mus-kitchensink-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: 'kitchensink', namespace: 'mus-kitchensink-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-dev', svcName: 'kitchensink', verbose: 'false'
  }

  stage('Integration Test') {
    // TBD: Proper test
    // Could use the OpenShift-Tasks REST APIs to make sure it is working as expected.

    //def newTag = "ProdReady-${version}"
    def newTag = "StagingCandidate-${version}"
    echo "New Tag: ${newTag}"

    // Replace xyz-tasks-dev with the name of your dev project
    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  stage('Deploy to Staging'){
    sh "oc project mus-kitchensink-staging"
    sh "oc patch dc kitchensink --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"kitchensink\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:StagingCandidate-$version\"}}}]}}' -n mus-kitchensink-stage"

    openshiftDeploy depCfg: 'kitchensink', namespace: 'mus-kitchensink-stage', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: 'kitchensink', namespace: 'mus-kitchensink-stage', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-stage', svcName: 'kitchensink', verbose: 'false'

  }

  stage('Build Prod Tag'){
    def newTag = "ProdReady-${version}"
    echo "New Tag: ${newTag}"

    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  def dest   = "tasks-green"
  def active = ""

  stage('Prep Production Deployment') {
    // Replace xyz-tasks-dev and xyz-tasks-prod with
    // your project names
    sh "oc project mus-kitchensink-prod"
    sh "oc get route tasks -n mus-kitchensink-prod -o jsonpath='{ .spec.to.name }' > activesvc.txt"
    active = readFile('activesvc.txt').trim()
    if (active == "tasks-green") {
      dest = "tasks-blue"
    }
    echo "Active svc: " + active
    echo "Dest svc:   " + dest
  }
  stage('Deploy new Version') {
    echo "Deploying to ${dest}"

    // Patch the DeploymentConfig so that it points to
    // the latest ProdReady-${version} Image.
    // Replace xyz-tasks-dev and xyz-tasks-prod with
    // your project names.
    sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:ProdReady-$version\"}}}]}}' -n mus-kitchensink-prod"

    openshiftDeploy depCfg: dest, namespace: 'mus-kitchensink-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: dest, namespace: 'mus-kitchensink-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-prod', svcName: dest, verbose: 'false'
  }
  stage('Switch over to new Version') {
    input "Switch Production?"

    // Replace xyz-tasks-prod with the name of your
    // production project
    sh 'oc patch route tasks -n mus-kitchensink-prod -p \'{"spec":{"to":{"name":"' + dest + '"}}}\''
    sh 'oc get route tasks -n mus-kitchensink-prod > oc_out.txt'
    oc_out = readFile('oc_out.txt')
    echo "Current route configuration: " + oc_out
  }
}

// Convenience Functions to read variables from the pom.xml
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
