# INDY CI jenkins store

This is Jenkins CI script store for indy product and library

## User Guide

Designed to manage NOS team's git repository and artifact in an automated way.

* `release-master.Jenkinsfile` is designed to **manage GitHub repo and sonatype artifact**, and schedule build parameter in order to test release version with various tests.
* `indy-build.Jenkinsfile` is designed to **build and trigger test**, archive build artifact. control build step by changing parameter in jenkins pipeline.
* `librarybuid.Jenkinsfile` is designed to **build and trigger test for library of indy**, a common build pipeline for depenency of indy.

--------

`indy-build.Jenkinsfile` are versatile for various configuration\
e.g.
1. to set up auto build job for **master banch commit verification** , use Default Value set down below, and trigger by poll-scm.
2. for **nightly build**, use build periodically and enable `FUNCTIONAL_TEST:true`<sub>boolean</sub>, `STRESS_TEST:true`<sub>boolean</sub>. to detect performance nightly.
3. to build artifact for **release** in maven central use `INDY_GIT_BRANCH:release` `INDY_MAJOR_VERSION:nextversion` `INDY_DEV_IMAGE_TAG:latest-release` `QUAY_TAG:latest-release`. and pipeline will archive all artifact needed for upload to sonatype.
4. setting for **pull request** verification, change setting like `INDY_GIT_BRANCH:or origin/pull/1507/head` and turn `FORCE_PUBLISH_IMAGE:false`<sub>boolean</sub>

--------

To setup an complete release sequence using `release-master.Jenkins`
1. you **must** set up indy related buid job first, e.g _indy-playground_.
2. and you **must** set up a Jenkins Credential of GitHub bot account for commit

job name in trigger job of release-master is not critical, if job name has changed, change it as well.\
to release from master branch, use `INDY_GIT_BRANCH:master` in release-master.Jenkinsfile.

--------

Set up performance benchmark by `Autotest.Jenkinsfile`, see parameter requirement downbelow.\
This is a critical step for CI/CD, and a requirement if `STRESS_TEST` set to be `true`.

*The `STRESS_TEST` Jenkins job requirement is not limited to `Autotest.Jenkinsfile`, any equivalent Jenkins job meets requirement will work*

## Jenkins parameters requirement

**indy-build required**

|Parameters      |Type |Default Value                                          |
|----------------|-----|-------------------------------------------------------|
|JENKINS_AGENT_CLOUD_NAME|String|openshift|
|INDY_GIT_BRANCH|String|master|
|INDY_GIT_REPO|String|https://github.com/Commonjava/indy|
|INDY_MAJOR_VERSION|String|2.0.0|
|INDY_PREPARE_RELEASE|Boolean|false|
|INDY_IMAGESTREAM_NAME|String|indy_binary|
|INDY_IMAGESTREAM_NAMESPACE|String|nos-automation|
|INDY_DEV_IMAGE_TAG|String|latest|
|FORCE_PUBLISH_IMAGE|Boolean|false|
|TAG_INTO_IMAGESTREAM|Boolean|true|
|MAIL_ADDRESS|String|liyu@redhat.com|
|TOWER_HOST|String|''|
|TOWER_TEMPLATE_NUMBER|String|850|
|FUNCTIONAL_TEST|Boolean|true|
|STRESS_TEST|Boolean|true|
|QUAY_IMAGE_TAG|String|latest|


* `JENKINS_AGENT_CLOUD_NAME` should be kubernetes plugin cluser name
* `INDY_GIT_BRANCH` can also be git commit reference e.g.:commit id or origin/pull/1507/head
* `INDY_MAJOR_VERSION` is optional, only needed when release a new version e.g. building release artifact.
* `TOWER_HOST` is the jenkins tower used to deploy a test envrioment.
* `TOWER_TEMPLATE_NUMBER` 850 is templpate id of `nos-automation - deploy-indy-perf`.
* Jenkins Credential Username and Password `Tower_Auth` is needed and script can access it.
* Skipp maven functional test and jmeter stress test by disable `FUNCTIONAL_TEST` and `STRESS_TEST`.
* `QUAY_IMAGE` tag must be latest in order to run `STRESS_TEST`, it's needed by ansible to deploy.

**release-master required**

|Parameters      |Type |Default Value                                          |
|----------------|-----|-------------------------------------------------------|
|JENKINS_AGENT_CLOUD_NAME|String|openshift|
|INDY_GIT_BRANCH|String|master|
|INDY_GIT_REPO|String|https://github.com/Commonjava/indy|
|INDY_MAJOR_VERSION|String|2.0.0|
|MAIL_ADDRESS|String|liyu@redhat.com|
|BOT_EMAIL|String|*@Commonjava.org|

* `INDY_GIT_BRANCH` should be maint or master etc, but proceed with careful, as it perform merge and push.
* Credential `GitHub_Bot` in Jenkins is GitHub account username and password, able to access `INDY_GIT_BRANCH`.

**Autotest required**

|Parameters      |Type |Default Value                                          |
|----------------|-----|-------------------------------------------------------|
|JENKINS_AGENT_CLOUD_NAME|String|openshift|
|THREADS|String|5|
|INDY_HOSTNAME|String|indy-perf.example.com|
|LOOPS|String|10|

* `THREADS` controls concurrent client simulation, and `LOOPS` controls total time of stress test.
* `INDY_HOSTNAME` should be where indy have been deployed, **MUST NOT** include protocal like `http://` or `https://`