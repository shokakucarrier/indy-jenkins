pipeline {
    agent { label 'maven' }
    stages {
        stage('git checkout') {
            checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                      extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy'], [$class: 'CleanCheckout']],
                      submoduleCfg: [], userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])
        }
        stage('Get Version'){
            steps{
                sh """# /bin/bash
                ehco 'Executing build for : ${params.LIB_GIT_REPO} ${params.LIB_MAJOR_VERSION}:${BUILD_NUMBER}'
                cd indy
                mvn versions:set -DnewVersion=${params.LIB_MAJOR_VERSION}:rc${BUILD_NUMBER}
                '"""
            }
        }
        stage('Build'){
            steps{
                sh "mvn -B -V clean verify"
            }
        }
        stage('Function test'){
            steps {
                sh 'mvn -B -V verify -Prun-its -Pci'
            }
        }
        stage('Archive') {
            steps {
                echo "Archive"
                archiveArtifacts artifacts: "**/*${params.LIB_MAJOR_VERSION}:rc${BUILD_NUMBER}*", fingerprint: true
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploy"
                sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
            }
        }
    }
}