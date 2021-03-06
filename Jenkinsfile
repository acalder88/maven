#!/usr/bin/env groovy
@Library('xtiva-utils')
import com.xtiva.jenkins.Utilities
import com.xtiva.jenkins.projects.Maven

node {
  def utils = new Utilities(this)
  def project = new Maven(this)
  def shouldRunSonar = utils.shouldRunSonar()
  currentBuild.result = "SUCCESS"
  def version, snapshot, image
  try {
    stage("Setup"){
      checkout scm
      version = project.version
      snapshot = project.snapshot
      def repo  = utils.getRepoName()
      echo("Building ${repo}: ${version}-${snapshot}")
    }
    stage("Validations"){
      utils.validateBranchName()
      utils.validateBranchLength()
    }
    stage("Test"){
      if(shouldRunSonar) {
        project.runInstrumentation()
      }else{
        project.runUnitTests()
      }
      utils.saveTestsResults('**/target/**/TEST-*.xml')
    }
    stage("Sonar"){
      if(!shouldRunSonar) {
        echo("This branch is not a pull request or develop. No need for analysis.")
      }else{
        //utils.runSonar(version, snapshot);
      }
    }
    stage("Build"){
      project.run("clean package -DskipTests")
      def repo = "development/poc-cd"
      def source = "poc-cd*.jar"
      def destination = "poc-cd.jar"
      image = project.buildImage(version, snapshot, repo, source, destination)
    }
    stage("Publish"){
      def path = "s3://xtiva-s3-versioning/01-Development/poc-cd/${version}-${snapshot}"
      if(snapshot == "" || snapshot == "master") {
        path = "s3://xtiva-s3-versioning/01-Development/poc-cd/${version}"
      }
      sh("aws s3 cp ./ ${path}/ --recursive --exclude '*' --include '*.template' --include '*.repo' --include '*.json' --include '*.sh'  --include '*.service' --include '*.timer' --include '*.yml'")
      utils.publishImage(image)
    }
    stage("Deploy") {
      def descriptor = 'poc-cd-marathon.json'
      def deployed = false
      sh("sed -i \"s/{{TAG}}/${image.tag}/g\" ${descriptor}")
      String id = sh(script:"cat ${descriptor} | jq -r '.id'", returnStdout:true).trim()
      def groups = sh(script: "dcos marathon group list |awk '{print\$1}'", returnStdout:true).trim().split("\n")
      echo ("${groups}")
      echo("${id}")
      for (String group : groups) {
        def areEquals = (id.equals(group))
        if (id.equals(group)) {
          deployed = true
        }
      }
      if(!deployed) {
        sh("dcos marathon group add ${descriptor}")
      } else {
        sh("dcos marathon group remove --force ${id}")
        sh("dcos marathon group add ${descriptor}")
      }
    }
  } catch (error) {
    currentBuild.result = "FAILURE"
    echo "project build error: ${error}"
    if (env.BRANCH_NAME == 'develop') {
      utils.sendEmail(this)
    }
    throw error
  }
}
