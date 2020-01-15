pipeline {
  agent{
    kubernetes {
      cloud params.JENKINS_AGENT_CLOUD_NAME
      label "jenkins-slave-${UUID.randomUUID().toString()}"
      serviceAccount "jenkins"
      defaultContainer 'jnlp'
      yaml """
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: "jenkins-${env.JOB_BASE_NAME}"
      spec:
        containers:
        - name: jnlp
          image: quay.io/kaine/indy-stress-tester:latest
          imagePullPolicy: Always
          tty: true
          - name: HOME
            value: /home/jenkins
          resources:
            requests:
              memory: 1500Mi
              cpu: 2000m
            limits:
              memory: 3000Mi
              cpu: 4000m
      """
    }
  }
  stages {
    stage('Run Build Test'){
      steps {
        script {
          echo "Jmeter running build-simulation-existing"
          sh script: "THREAD=${params.THREADS} HOSTNAME=${params.INDY_HOSTNAME} LOOPS=${params.LOOPS} PORT=80 /src/entrypoint.sh build-simulation-existing.jmx"
        }
      }
    }
    stage('Run Download Test'){
      steps {
        script {
          echo "Jmeter running download-simulation-existing"
          sh script: "THREAD=${params.THREADS} HOSTNAME=${params.INDY_HOSTNAME} LOOPS=${params.LOOPS} PORT=80 /src/entrypoint.sh download-simulation-existing.jmx"
        }
      }
    }
    stage('Run Upload Test'){
      steps {
        script {
          echo "Jmeter running upload-simulation-existing"
          sh script: "THREAD=${params.THREADS} HOSTNAME=${params.INDY_HOSTNAME} LOOPS=${params.LOOPS} PORT=80 /src/entrypoint.sh upload-simulation-existing.jmx"
        }
      }
    }
    stage('Archive & Publish'){
      steps{
        script{
          sh script: "cp /src/*.log ./"
          sh script: """#!/bin/bash
          echo "<?xml version=\\"1.0\\" encoding=\\"UTF-8\\"?>" >> combined.xml && \
          echo "<testResults version=\\"1.2\\">" >> combined.xml && \
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