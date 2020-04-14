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
          image: quay.io/kaine/indy-stress-tester:latest
          imagePullPolicy: Always
          tty: true
          env:
          - name: USER
            value: 'jenkins-k8s-config'
          - name: HOME
            value: /home/jenkins
          resources:
            requests:
              memory: 1Gi
              cpu: 1000m
            limits:
              memory: 1Gi
              cpu: 1000m
          volumeMounts:
          - mountPath: /home/jenkins/sonatype
            name: volume-0
          - mountPath: /home/jenkins/gnupg_keys
            name: volume-1
          - mountPath: /mnt/ocp
            name: volume-2
          workingDir: /home/jenkins
        volumes:
        - name: volume-0
          secret:
            defaultMode: 420
            secretName: sonatype-secrets
        - name: volume-1
          configMap:
            defaultMode: 420
            name: gnupg
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
  }
  stages {
    stage('git checkout') {
      steps{
        script{
          checkout([$class      : 'GitSCM', branches: [[name: params.LIB_GIT_BRANCH]], doGenerateSubmoduleConfigurations: false,
                    extensions  : [[$class: 'RelativeTargetDirectory', relativeTargetDir: params.LIB_NAME]], submoduleCfg: [],
                    userRemoteConfigs: [[url: params.LIB_GIT_REPO]]])

          echo "Prepare the release of ${params.LIB_GIT_REPO} branch: ${params.LIB_GIT_BRANCH}"

          sh """#!/bin/bash
          cd ${params.LIB_NAME}
          git checkout ${params.LIB_GIT_BRANCH}
          gpg --allow-secret-key-import --import /home/jenkins/gnupg_keys/private_key.txt
          """
        }
      }
    }
    stage('Setup and Formating'){
      steps{
        script{
          withCredentials([
            usernamePassword(credentialsId:'GitHub-Bot', passwordVariable:'BOT_PASSWORD', usernameVariable:'BOT_USERNAME'),
            usernamePassword(credentialsId:'OSS-Nexus-Bot', passwordVariable:'OSS_BOT_PASSWORD', usernameVariable:'OSS_BOT_USERNAME'),
            string(credentialsId: 'gnupg_passphrase', variable: 'PASSPHRASE')
          ]) {
            dir(params.LIB_NAME){
              sh """
              git config --global user.email "${params.BOT_EMAIL}"
              git config --global user.name "${BOT_USERNAME}"
              mvn -B -s ../settings.xml -Pformatting clean install
              git commit -a, "Update license header"                #commit nothing when there is no file needs to be modified
              git push https://${BOT_USERNAME}:${BOT_PASSWORD}@$`python3 -c 'print("${params.LIB_GIT_REPO}".split("//")[1])'` --all
              cp ../settings-release.xml /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSPHRASE}}/${PASSPHRASE}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_USERNAME}}/${OSS_BOT_USERNAME}/g' /home/jenkins/.m2/settings.xml
              sed -i 's/{{_PASSWORD}}/${OSS_BOT_PASSWORD}/g' /home/jenkins/.m2/settings.xml
              """
            }
          }
        }
      }
    }
    stage('Release Prepare'){
      steps{
        script{
          dir(params.LIB_NAME){
            env.LIB_NEXT_VERSION = sh (
              script: """
                echo ${params.LIB_MAJOR_VERSION} | awk -F. -v OFS=. 'NF==1{print ++$NF}; NF>1{if(length($NF+1)>length($NF))$(NF-1)++; $NF=sprintf("%0*d", length($NF), ($NF+1)%(10^length($NF))); print}'
                """
              returnStdout: true
            )
            sh """#!/bin/bash
            mvn help:effective-settings
            mvn --batch-mode release:prepare -DreleaseVersion=${params.LIB_MAJOR_VERSION} -DdevelopmentVersion=${env.LIB_NEXT_VERSION}-SNAPSHOT -Dtag=indy-parent-${params.LIB_MAJOR_VERSION}
            """
          }
        }
      }
    }
    stage('Release performing'){
      steps{
        script{
          dir(params.LIB_NAME){
            sh """
            mvn --batch-mode release:perform
            """
          }
        }
      }
    }
  }
  post {
    cleanup {
      script{
        if (params.INDY_GIT_BRANCH == 'release' || params.INDY_PREPARE_RELEASE == true){
          try{
            sh """
            curl -X DELETE "http://indy-infra-nos-automation.cloud.paas.psi.redhat.com/api/admin/stores/maven/hosted/${params.INDY_MAJOR_VERSION}-jenkins-${env.BUILD_NUMBER}" -H "accept: application/json"
            """
          }catch(e){
            echo "Error teardown hosted repo"
          }
        }
        if (env.RESULTING_TAG) {
          echo "Removing tag ${env.RESULTING_TAG} from the ImageStream..."
          openshift.withCluster() {
            openshift.withProject("${params.INDY_IMAGESTREAM_NAMESPACE}") {
              openshift.tag("${params.INDY_IMAGESTREAM_NAME}:${env.RESULTING_TAG}",
                "-d")
            }
          }
        }
      }
    }
    success {
      script {
        echo "SUCCEED"
      }
    }
    failure {
      script {
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

def sendBuildStatusEmail(String status) {
  def recipient = params.MAIL_ADDRESS
  def subject = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} ${status}."
  def body = "Build URL: ${env.BUILD_URL}"
  if (env.PR_NO) {
    subject = "Jenkins job ${env.JOB_NAME}, PR #${env.PR_NO} ${status}."
    body += "\nPull Request: ${env.PR_URL}"
  }
  emailext to: recipient, subject: subject, body: body
}