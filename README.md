# INDY CI jenkins store

This is Jenkins CI script store for indy product and library

* basebuild.Jenkinsfile : build and functian test for indy launcher and Library packed within

## Jenkins files

**indy-build required**

|Parameters      |Type |Default Value                                          |
|----------------|-----|-------------------------------------------------------|
|JENKINS_AGENT_CLOUD_NAME|String|openshift|
|INDY_GIT_BRANCH|String|master|
|INDY_GIT_REPO|String|https://github.com/Commonjava/indy|
|INDY_MAJOR_VERSION|String|2.0.0|
|INDY_IMAGESTREAM_NAME|String|indy_binary|
|INDY_IMAGESTREAM_NAMESPACE|String|nos-automation|
|INDY_DEV_IMAGE_TAG|String|latest|
|FORCE_PUBLISH_IMAGE|Boolean|false|
|TAG_INTO_IMAGESTREAM|Boolean|true|
|MAIL_ADDRESS|String|liyu@redhat.com|
|TOWER_HOST|String|''|
|TOWER_TEMPLATE_NUMBER|String|850|

_indy branch can also be git commit reference_

_Jekins agent cloud name should be kubernetes plugin cluser name_

* Jenkins Credential Username and Password Tower_Auth is needed and script can access it.

_850 is templpate id of nos-automation - deploy-indy-perf_

**Autotest required**

|Parameters      |Type |Default Value                                          |
|----------------|-----|-------------------------------------------------------|
|THREADS|String|5|
|INDY_HOSTNAME|String|indy-perf-nos-automation.cloud.paas.psi.redhat.com|
|LOOPS|String|10|
|JENKINS_AGENT_CLOUD_NAME|String|openshift|