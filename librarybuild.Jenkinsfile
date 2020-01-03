pipeline {
    agent { label 'maven' }
    stages {
        stage('git checkout') {
            steps{
                checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                          extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.LIB_NAME], [$class: 'CleanCheckout']],
                          submoduleCfg: [], userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])
            }
        }
        stage('Get Version'){
            steps{
                sh """# /bin/bash
                echo 'Executing build for : ${params.LIB_GIT_REPO} ${params.LIB_MAJOR_VERSION}:${BUILD_NUMBER}'
                cd ${params.LIB_NAME}
                mvn versions:set -DnewVersion=${params.LIB_MAJOR_VERSION}-rc${BUILD_NUMBER}
                """
            }
        }
        stage('Build'){
            steps{
                dir(params.LIB_NAME){
                    sh "mvn -B -V clean verify"
                }
            }
        }
        stage('Function test'){
            steps {
                dir(params.LIB_NAME){
                    sh 'mvn -B -V verify -Prun-its -Pci'
                }
            }
        }
        stage('Archive') {
            steps {
                echo "Archive"
                archiveArtifacts artifacts: "**/*${params.LIB_MAJOR_VERSION}-rc${BUILD_NUMBER}*", fingerprint: true
            }
        }
        stage('Deploy') {
            steps {
                dir(params.LIB_NAME){
                    echo "Deploy"
                    sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
                }
            }
        }
    }
}