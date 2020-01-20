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
          env:
          - name: HOME
            value: /home/jenkins
          - name: USER
            value: 'jenkins-k8s-config'
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
    stage('run Ansible Tower template'){
      steps{
        withCredentials([usernamePassword(credentialsId:'Tower_Auth', passwordVariable:'PASSWORD', usernameVariable:'USERNAME')]) {
          sh """#!/bin/bash
          curl -u ${USERNAME}:${PASSWORD} \
          -H 'Content-Type: application/json' \
          --data '{}' \
          -X POST ${params.TOWER_HOST}api/v2/job_templates/${params.TEMPLATE_ID}/launch/
          """
        }
      }
    }
  }
}