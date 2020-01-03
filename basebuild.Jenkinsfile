def data_artifact="deployments/launcher/target/*-data.tar.gz"
def artifact="deployments/launcher/target/*-skinny.tar.gz"

def ocp_map = '/mnt/ocp/jenkins-openshift-mappings.json'
def bc_section = 'build-configs'
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
                mvn versions:set -DnewVersion=${params.INDY_MAJOR_VERSION}-rc${BUILD_NUMBER}
                """
            }
        }
        stage('Build'){
            steps{
                dir("indy"){
                    sh "mvn -B -V clean verify"
                }
            }
        }
        stage('Function test'){
            steps {
                dir("indy"){
                    sh 'mvn -B -V verify -Prun-its -Pci'
                }
            }
        }
        stage('Archive') {
            steps {
                echo "Archive"
                archiveArtifacts artifacts: "**/*${params.INDY_MAJOR_VERSION}-rc${BUILD_NUMBER}*", fingerprint: true
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploy"
                dir("indy"){
                    sh 'mvn help:effective-settings -B -V -DskipTests=true deploy -e'
                }
            }
        }
        stage('Load OCP Mappings') {
            steps {
                echo "Load OCP Mapping document"
                script {
                    def exists = fileExists ocp_map
                    if (exists){
                        def jsonObj = readJSON file: ocp_map
                        if (bc_section in jsonObj){
                            if (params.INDY_GIT_URL in jsonObj[bc_section]) {
                                echo "Found BC for Git repo: ${params.INDY_GIT_URL}"
                                if (params.INDY_GIT_BRANCH in jsonObj[bc_section][params.INDY_GIT_URL]) {
                                    my_bc = jsonObj[bc_section][params.INDY_GIT_URL][params.INDY_GIT_BRANCH]
                                } else {
                                    my_bc = jsonObj[bc_section][params.INDY_GIT_URL]['default']
                                }

                                echo "Using BuildConfig: ${my_bc}"
                            }
                            else {
                                echo "Git URL: ${params.INDY_GIT_URL} not found in BC mapping."
                            }
                        }
                        else {
                            "BC mapping is invalid! No ${bc_section} sub-object found!"
                        }
                    }
                    else {
                        echo "JSON configuration file not found: ${ocp_map}"
                    }
                }
            }
        }
        stage('Build & Push Image') {
            steps {
                script {
                    dir("indy"){
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
}