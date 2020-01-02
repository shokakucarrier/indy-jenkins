def data_artifact="deployments/launcher/target/*-data.tar.gz"

def my_bc = null

pipeline {
    agent { label 'maven' }
    stages {
        stage('git checkout') {
            steps{
                checkout([$class      : 'GitSCM', branches: [[name: params.INDY_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                          extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy'], [$class: 'CleanCheckout']],
                          submoduleCfg: [], userRemoteConfigs: [[url: params.INDY_GIT_REPO]]])
            }
        }
        stage('Get Version'){
            steps{
                sh """# /bin/bash
                echo 'Executing build for : ${params.INDY_GIT_REPO} ${params.INDY_MAJOR_VERSION}:${BUILD_NUMBER}'
                cd indy
                mvn versions:set -DnewVersion=${params.INDY_MAJOR_VERSION}:rc${BUILD_NUMBER}
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
                archiveArtifacts artifacts: "**/*${params.INDY_MAJOR_VERSION}:rc${BUILD_NUMBER}*", fingerprint: true
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploy"
                sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
            }
        }
        stage('Build & Push Image') {
            steps {
                script {
                    def artifact_file = sh(script: "ls $artifact", returnStdout: true)?.trim()
                    def tarball_url = "${BUILD_URL}artifact/$artifact_file"
                    openshift.withCluster() {
                        openshift.withProject() {
                            echo "Starting image build: ${openshift.project()}:${my_bc}"
                            def bc = openshift.selector("bc", my_bc)

                            def data_artifact_file = sh(script: "ls $data_artifact", returnStdout: true)?.trim()
                            def data_tarball_url = "${BUILD_URL}artifact/$data_artifact_file"
                            
                            def buildSel = bc.startBuild("-e tarball_url=${tarball_url} -e data_tarball_url=${data_tarball_url}")
                            buildSel.logs("-f")
                        }
                    }
                }
            }
        }
    }
}