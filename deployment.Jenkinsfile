pipeline {
    agent { label 'indystress' }
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