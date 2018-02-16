#!groovy

// Run this node on a Maven Slave
// Maven Slaves have JDK and Maven already installed
node('maven') {
  // Make sure your nexus_openshift_settings.xml
  // Is pointing to your nexus instance
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  // Checkout the source code and create a tag
  stage('Checkout Source') {
    // The Jenkins BuildConfig in OCP points to a Jenkinsfile which is this file
    // This file lives in the root of the gogs repository that the repo in the
    // BC points to.
    checkout scm
    // Now we need to set up git configuration
    sh("git config --global user.email 'mmusaji@redhat.com'")
    sh("git config --global user.name 'Mustafa Musaji'")
    // Get the version number from Jenkins and create a tag from it
    // This is one way to ensure you can track images from build through to
    // what is running on a production instance as the image is built once based
    // off this codebase and then tagged with the same version number.
    sh("git tag -a 7.0.'${currentBuild.number}' -m 'Jenkins'")
    // Push the tag
    sh('git push http://mmusaji:redhat@gogs-demo-mus-cicd.apps.2245.openshift.opentlc.com/mmusaji/kitchensink.git --tags')
  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  // Make sure we set the version number to the same as the Jenkins build no.
  def version    = "7.0.${currentBuild.number}"

  // Build the source
  stage('Build war') {
    echo "Building version ${version}"

    // Use maven commands to set the version number and not the one in the pom
    sh "${mvnCmd} versions:set -DnewVersion=7.0.${version}"
    sh "${mvnCmd} clean package -DskipTests -Popenshift"
  }

  // Carry out the Unit Tests
  stage('Unit Tests') {
    echo "Unit Tests"
  //  sh "${mvnCmd} test"
  }

  // Carry out code coverage analysis using SonarQube
  stage('Code Analysis') {
    echo "Code Analysis"

    //sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-mus-cicd.apps.2245.openshift.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME}"
  }

  // Publish the created the artifact to our internal nexus repo
  stage('Publish to Nexus') {
    echo "Publish to Nexus"

    // If you try and deploy a SNAPSHOT release to the releases endpoint Nexus
    // won't allow you. So you could check here if it's SNAPSHOT release to push
    // to another repo for different build. In this case, we know we are
    // attempting full CI/CD example so we want to push a build with a version
    // as soon as new code is checked in to the repository
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.mus-cicd.svc.cluster.local:8081/repository/releases"
  }

  //Using the artifact build the OpenShift Image we are going to deploy
  stage('Build OpenShift Image') {
    // Lets create a Tag that the Dev Env will deploy
    def newTag = "TestingCandidate-${version}"
    echo "New Tag: ${newTag}"

    // Copy the war file we just built and rename to ROOT.war
    sh "cp ./target/jboss-kitchensink-angularjs.war ./ROOT.war"

    // Start Binary Build in OpenShift using the file we just published
    // Quick check for debug purposes
    sh "oc whoami"

    //Switch to the right project
    sh "oc project mus-kitchensink-dev"

    // Start the build from the BC we created as part of the set up stages...
    // [RAN AS PART OF SETUP]
    // "oc new-build --binary=true --name="kitchensink" jboss-eap70-openshift:1.5"
    sh "oc start-build kitchensink --follow --from-file=./deployments/ROOT.war -n mus-kitchensink-dev"

    //Once build completes we want to tag this image with the "TestingCandidate-${version}"
    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  //Deploy this to image to Dev environments
  stage('Deploy to Dev') {
    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
    // This way of doing this is a little weird. Real world you would probably
    // want to wait for the latest image change and just deploy, and then tag
    // a version that you want to deploy to production.
    sh "oc project mus-kitchensink-dev"

    //Patch the DC so it ImageChange is looking for the name TestingCandidate-$version
    sh "oc patch dc kitchensink --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"kitchensink\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:TestingCandidate-$version\"}}}]}}' -n mus-kitchensink-dev"

    // Do a deployment
    openshiftDeploy depCfg: 'kitchensink', namespace: 'mus-kitchensink-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'

    //Verify the deployment and the Services is live
    openshiftVerifyDeployment depCfg: 'kitchensink', namespace: 'mus-kitchensink-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-dev', svcName: 'kitchensink', verbose: 'false'
  }

  // Our integration tests doesn't actually do anything at the moment
  // However, it shows how you take an image from a previous stage without having
  // to do another mvn build which is exactly what you don't want to do
  stage('Integration Test') {
    // TBD: Proper test
    // Could use the OpenShift-Tasks REST APIs to make sure it is working as expected.

    def newTag = "StagingCandidate-${version}"
    echo "New Tag: ${newTag}"

    //Integration tests Passed? So let's tag this as StagingCandidate release
    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  //Another stage that doesn't do anything but is purely for staging
  stage('Deploy to Staging'){

    sh "oc project mus-kitchensink-stage"

    // Patch the DC so that we look for a staging image we can deploy
    sh "oc patch dc kitchensink --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"kitchensink\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:StagingCandidate-$version\"}}}]}}' -n mus-kitchensink-stage"

    openshiftDeploy depCfg: 'kitchensink', namespace: 'mus-kitchensink-stage', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: 'kitchensink', namespace: 'mus-kitchensink-stage', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-stage', svcName: 'kitchensink', verbose: 'false'

  }

  //Now build the prod tag
  stage('Build Prod Tag'){
    def newTag = "ProdReady-${version}"
    echo "New Tag: ${newTag}"

    //Tag this as ProdReady
    openshiftTag alias: 'false', destStream: 'kitchensink', destTag: newTag, destinationNamespace: 'mus-kitchensink-dev', namespace: 'mus-kitchensink-dev', srcStream: 'kitchensink', srcTag: 'latest', verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  def dest   = "tasks-green"
  def active = ""

  //Prep the Prod Deployment by getting which route is currently active
  stage('Prep Production Deployment') {

    sh "oc project mus-kitchensink-prod"
    sh "oc get route tasks -n mus-kitchensink-prod -o jsonpath='{ .spec.to.name }' > activesvc.txt"
    active = readFile('activesvc.txt').trim()
    if (active == "tasks-green") {
      dest = "tasks-blue"
    }
    echo "Active svc: " + active
    echo "Dest svc:   " + dest
  }

  // Now we deploy the image
  stage('Deploy new Version') {
    echo "Deploying to ${dest}"

    // Patch the DeploymentConfig so that it points to
    // the latest ProdReady-${version} Image.
    sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mus-kitchensink-dev\", \"name\": \"kitchensink:ProdReady-$version\"}}}]}}' -n mus-kitchensink-prod"

    openshiftDeploy depCfg: dest, namespace: 'mus-kitchensink-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: dest, namespace: 'mus-kitchensink-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mus-kitchensink-prod', svcName: dest, verbose: 'false'
  }

  // But the route still points to the old version so we have to switch it
  stage('Switch over to new Version') {
    //Get approval!
    input "Switch Production?"

    // Patch the route to switch to the new once
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
