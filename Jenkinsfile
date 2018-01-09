pipeline {

  agent {
    node {
      label 'docker.ci.uktrade.io'
    }
  }

  stages {
    stage('prep') {
      steps {
        script {
          sh 'date +"%Y%m%d.%H%M%S" > dateFile'
          env.BUILD_VERSION = readFile ('dateFile').trim()
          deployer = docker.image("ukti/lite-image-builder")
          deployer.pull()
        }
      }
    }

    stage('build') {
      steps {
        script {
          deployer.inside {
            sh 'echo $BUILD_VERSION'
            
            if (env.BuildType.toLowerCase() == 'jar'){
                echo "running ${BuildType} Build Type"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/${Branch}']], 
                    userRemoteConfigs: [[url: 'git@github.com:uktrade/${ProjectName}.git']],
                    credentialsId: env.SCM_CREDENTIAL
                ])
                sh 'chmod 777 gradlew'
                sh "./gradlew -PprojVersion=${BUILD_VERSION} :publishServicePublicationToLite-buildsRepository"
            }
            
            if (env.BuildType.toLowerCase() == 'zip') {
                echo "running ${BuildType} Build Type"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/${Branch}']], 
                    userRemoteConfigs: [[url: 'git@github.com:uktrade/${ProjectName}.git']],
                    credentialsId: env.SCM_CREDENTIAL,
                    extensions: [[$class: 'SubmoduleOption',
                        disableSubmodules: false,
                        parentCredentials: true,
                        recursiveSubmodules: true,
                        reference: '',
                        trackingSubmodules: false
                    ]]
                ])
                sh 'printf "[repositories]\\n  local\\n  my-ivy-proxy-releases: https://nexus.ci.uktrade.io/repository/ivy-proxy/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]\\n  my-maven-proxy-releases: https://nexus.ci.uktrade.io/repository/lite-build-and-proxy/" > ~/.sbt/repositories'
                sh 'cat ~/.sbt/repositories'
                sh 'ls -l subprojects/lite-play-common'
                sh 'sbt -Dsbt.override.build.repos=true publish'
            }
          }  
        }
      }
    }

    stage('Git tag') {
        steps {
            sshagent([env.SCM_CREDENTIAL]) {
              sh "git clone git@github.com:uktrade/${ProjectName}.git"   
              sh "git -c 'user.name=Jenkins' -c 'user.email=jenkins@digital' tag  -a ${BUILD_VERSION} -m 'Jenkins'"
              sh "git push --tags"
            }
        }
    }

    stage ('Invoke_ci-pipeline'){
        steps { 
            sh 'echo $BUILD_VERSION'
            sh 'echo $Environment'
            build job: 'ci-pipeline', parameters: [
                    string(name: 'Team', value: 'lite'), 
                    string(name: 'Project', value: env.ProjectName), 
                    string(name: 'Environment', value: env.Environment), 
                    string(name: 'Version', value: env.BUILD_VERSION) 
                ]
        }
    }
  }

  post {
    always {
      script {
        currentBuild.displayName = "#" + currentBuild.number + " - Tag: " + env.BUILD_VERSION  
        deleteDir()
      }
    }
  }
  
}
