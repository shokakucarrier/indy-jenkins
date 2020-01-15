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
          - mountPath: /home/jenkins/.m2
            name: volume-1
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
        - name: volume-1
          secret:
            defaultMode: 420
            secretName: maven-secrets
        - name: volume-2
          configMap:
            defaultMode: 420
            name: jenkins-openshift-mappings
      """
    }
  }
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
        echo 'Executing build for : ${params.LIB_GIT_REPO} ${params.LIB_MAJOR_VERSION}-rc${BUILD_NUMBER}'
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
    stage('Functional test'){
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