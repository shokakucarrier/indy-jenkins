def userInput = []
def inputThreads = null
def inputLoops = null
def inputUrl = null
def inputPort = null

pipeline {
    agent {
      node {label 'indystress'}
    }
    stages {
        stage('Enter Parameters'){
            steps {
                script {
                    userInput = input(
                        id: 'userInput', message: "Please enter a test suite to run:",
                        parameters:[
                            string(name: 'threads', defaultValue: '5', description: 'Jmeter treads'),
                            string(name: 'loops', defaultValue: '10', description: 'Script loops, controls running time'),
                            string(name: 'url', defaultValue: 'indy-perf-nos-automation.cloud.paas.psi.redhat.com', description: 'Indy URL to test'),
                            string(name: 'port', defaultValue: '80', description: 'Indy port to test')
                        ]
                    )
                    inputThreads = userInput.threads?:'5'
                    inputLoops = userInput.loops?:''
                    inputUrl = userInput.url?:''
                    inputPort = userInput.port?:'80'
                }
            }
        }
        stage('Run Build Test'){
            steps {
                script {
                    echo "Jmeter running build-simulation-existing"
                    sh script: "THREADS=${inputThreads} HOSTNAME=${inputUrl} LOOPS=${inputLoops} PORT=${inputPort} /src/entrypoint.sh build-simulation-existing.jmx"
                }
            }
        }
        stage('Run Download Test'){
            steps {
                script {
                    sh script: "cp -R ./inputs/properties/* /src/inputs/properties/"
                    echo "Jmeter running download-simulation-existing"
                    sh script: "THREADS=${inputThreads} HOSTNAME=${inputUrl} LOOPS=${inputLoops} PORT=${inputPort} /src/entrypoint.sh download-simulation-existing.jmx"
                }
            }
        }
        stage('Run Upload Test'){
            steps {
                script {
                    sh script: "cp -R ./inputs/properties/* /src/inputs/properties/"
                    echo "Jmeter running upload-simulation-existing"
                    sh script: "THREADS=${inputThreads} HOSTNAME=${inputUrl} LOOPS=${inputLoops} PORT=${inputPort} /src/entrypoint.sh upload-simulation-existing.jmx"
                }
            }
        }
        stage('Archive & Publish'){
            steps{
                script{
                    sh script: "cp /src/*.log $WORKSPACE"
                    sh script: """#!/bin/bash
                    echo "<?xml version=\"1.0\" encoding=\"UTF-8\"?>" >> combined.xml && \
                    echo "<testResults version=\"1.2\">" >> combined.xml && \
                    grep -vh "</\\?testResults>\\|<?xml\\|<testResults" build-simulation-existing.jmx.log >> combined.xml && \
                    grep -vh "</\\?testResults>\\|<?xml\\|<testResults" upload-simulation-existing.jmx.log >> combined.xml && \
                    grep -vh "</\\?testResults>\\|<?xml\\|<testResults" download-simulation-existing.jmx.log >> combined.xml && \
                    echo "</testResults>" >> combined.xml
                    """
                }
                archiveArtifacts artifacts: "*.log,combined.xml"
                perfReport "combined.xml"
            }
        }
    }
}