pipeline {
  agent {
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
          indy-pipeline-build-number: "${env.BUILD_NUMBER}"
      spec:
        containers:
        - name: jnlp
          image: registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11
          imagePullPolicy: Always
          tty: true
          env:
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx4g'
          - name: MAVEN_OPTS
            value: '-Xmx8g -Xms1024m -XX:MaxPermSize=512m -Xss8m'
          - name: USER
            value: 'jenkins-k8s-config'
          - name: IMG_BUILD_HOOKS
            valueFrom:
              secretKeyRef:
                key: img-build-hooks.json
                name: img-build-hooks-secrets
          - name: HOME
            value: /home/jenkins
          resources:
            requests:
              memory: 4Gi
              cpu: 2000m
            limits:
              memory: 8Gi
              cpu: 4000m
          volumeMounts:
          - mountPath: /home/jenkins/sonatype
            name: volume-0
          - mountPath: /mnt/ocp
            name: volume-2
          workingDir: /home/jenkins
        volumes:
        - name: volume-0
          secret:
            defaultMode: 420
            secretName: sonatype-secrets
        - name: volume-2
          configMap:
            defaultMode: 420
            name: jenkins-openshift-mappings
      """
    }
  }
  options {
    //timestamps()
    timeout(time: 120, unit: 'MINUTES')
  }
  environment {
    PIPELINE_NAMESPACE = readFile('/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
    PIPELINE_USERNAME = sh(returnStdout: true, script: 'id -un').trim()
    GITHUB_API_URL='https://api.github.com/repos/Commonjava/indy'
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          checkout([$class      : 'GitSCM', branches: [[name: params.INDY_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'indy'], [$class: 'CleanCheckout']], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.INDY_GIT_REPO, refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*/head:refs/remotes/origin/pull/*/head']]])

          echo "Prepare the release of ${params.INDY_GIT_REPO} branch: ${params.INDY_GIT_BRANCH}"
        }
      }
    }
    stage('Get Version'){
      steps{
        dir('indy'){
          sh """#!/bin/bash
          mvn versions:set -DnewVersion=${params.INDY_MAJOR_VERSION}
          """
        }
      }
    }
    stage('Trigger build with same setting'){
      steps{
        build job: 'indy-playground', propagate: true, wait: true, parameters: [string(name: 'INDY_GIT_BRANCH', value: "${params.INDY_GIT_BRANCH}"),
        string(name: 'INDY_GIT_REPO', value: "${params.INDY_GIT_REPO}"),
        string(name: 'INDY_MAJOR_VERSION', value: "${params.INDY_MAJOR_VERSION}"),
        string(name: 'INDY_DEV_IMAGE_TAG', value: "latest"),
        booleanParam(name: 'TAG_INTO_IMAGESTREAM', value: true),
        booleanParam(name: 'INDY_PREPARE_RELEASE', value: true),
        booleanParam(name: 'FUNCTIONAL_TEST', value: true),
        booleanParam(name: 'STRESS_TEST', value: true),
        string(name: 'QUAY_IMAGE_TAG', value: 'latest')
        ]
      }
    }
    stage('Push tag&changes to release branch'){
      steps{
        script{
          withCredentials([usernamePassword(credentialsId:'GitHub-Bot', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME')]) {
            dir('indy'){
              sh """
              git config --global user.email "${params.BOT_EMAIL}"
              git config --global user.name "${BOT_USERNAME}"
              git commit -am "prepare release indy-parent-${params.INDY_MAJOR_VERSION}"
              git tag indy-parent-${params.INDY_MAJOR_VERSION}
              git checkout release
              git merge ${params.INDY_GIT_BRANCH}
              export HOSTNAME=`python3 -c 'print("${params.INDY_GIT_REPO}".split("//")[1])'`
              git push https://${BOT_USERNAME}:${BOT_PASSWORD}@${HOSTNAME} --all
              """
            }
          }
        }
      }
    }
    stage('trigger build of release branch'){
      steps{
        build job: 'indy-release', propagate: true, wait: true, parameters: [string(name: 'INDY_GIT_BRANCH', value: "release"),
        string(name: 'INDY_GIT_REPO', value: "${params.INDY_GIT_REPO}"),
        string(name: 'INDY_DEV_IMAGE_TAG', value: "latest-release"),
        booleanParam(name: 'TAG_INTO_IMAGESTREAM', value: true),
        booleanParam(name: 'FORCE_PUBLISH_IMAGE', value: true),
        booleanParam(name: 'INDY_PREPARE_RELEASE', value: false),
        booleanParam(name: 'FUNCTIONAL_TEST', value: false),
        booleanParam(name: 'STRESS_TEST', value: false),
        string(name: 'QUAY_IMAGE_TAG', value: 'latest-release')
        ]
      }
    }
    /*stage('release artifact to sonatype'){
      steps{
        script{

        }
      }
    }*/
  }
  post {
    success {
      script {
        echo "SUCCEED"
      }
    }
    failure {
      script {
        dir('indy'){
          sh 'git reset --hard'
          env.INDY_SNAPSHOT_DEPENDENCY = sh (
                  script: 'mvn -s ../settings.xml dependency:tree -Dincludes=:::*-SNAPSHOT | grep -v -e Downloaded -e Progress -e Downloading -e indy -e Indy -e "\\[ERROR\\]" -e "\\[\\ pom\\ \\]" -e "\\[\\ jar\\ \\]"',
                  returnStdout: true
            ).trim()
          if (params.MAIL_ADDRESS){
            try {
              sendBuildStatusEmail('failed')
            } catch (e) {
              echo "Error sending email: ${e}"
            }
          }
        }
      }
    }
  }
}

def sendBuildStatusEmail(String status) {
  def recipient = params.MAIL_ADDRESS
  def subject = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} ${status}."
  def body = "Build URL: ${env.BUILD_URL}"
  if (env.INDY_SNAPSHOT_DEPENDENCY) {
    body += "\nFound snapshot release depenency: \n ${INDY_SNAPSHOT_DEPENDENCY}"
  }
  if (env.PR_NO) {
    subject = "Jenkins job ${env.JOB_NAME}, PR #${env.PR_NO} ${status}."
    body += "\nPull Request: ${env.PR_URL}"
  }
  emailext to: recipient, subject: subject, body: body
}