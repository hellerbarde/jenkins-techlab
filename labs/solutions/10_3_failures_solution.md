Solution Lab 10: Failures and Notifications
===========================================

Solutions for [Lab 10.3: Mail notification](../10_failures.md#lab-103-mail-notification)

Extended version of Lab 10.1: Failures (Declarative Syntax)
-----------------------------------------------------------

Mail step documentation: <https://jenkins.io/doc/pipeline/steps/workflow-basic-steps/#mail-mail>.
The Snippet Generator in Jenkins is always a good help.

Create a new branch named lab-10.3 from branch lab-10.1 and change the contents of the Jenkinsfile to the following script with one change:

**Important:** Use your mail address in all ``mail to:`` commands instead of ``"me@ne.me"``.

```groovy
pipeline {
    agent { label env.JOB_NAME.split('/')[0] }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 10, unit: 'MINUTES')
        timestamps()  // Requires the "Timestamper Plugin"
    }
    triggers {
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Build') {
            steps {
                withEnv(["JAVA_HOME=${tool 'jdk11'}", "PATH+MAVEN=${tool 'maven35'}/bin:${env.JAVA_HOME}/bin"]) {
                    sh 'mvn -B -V -U -e clean verify -Dsurefire.useFile=false -Dmaven.test.failure.ignore=true'
                    archiveArtifacts 'target/*.?ar'
                }
            }
            post {
                always {
                    junit 'target/**/*.xml'  // Requires JUnit plugin
                }
            }
        }
    }
    post {
        success {
            mail to: "me@ne.me", subject: "Build success - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                 body: "Success of build: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>"
        }
        unstable {
            mail to: "me@ne.me", subject: "Build unstable - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                 body: "Unstable build: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>"
        }
        failure {
            mail to: "me@ne.me", subject: "Build failure - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                 body: "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>"
        }
    }
}
```
**Note:** Check your mail inbox.

Extended version of Lab 10.2: Failures (Scripted Syntax)
----------------------------------------------------------

Create a new branch named lab-10.4 from branch lab-10.1 and change the contents of the Jenkinsfile to the following script with one change:

**Important:** Use your mail address in all ``mail to:`` commands instead of ``"me@ne.me"``.

```groovy
properties([
    buildDiscarder(logRotator(numToKeepStr: '5')),
    pipelineTriggers([
        pollSCM('H/5 * * * *')
    ])
])

try {
    timestamps() {
        timeout(time: 10, unit: 'MINUTES') {
            node(env.JOB_NAME.split('/')[0]) {
                stage('Build') {
                    try {
                        withEnv(["JAVA_HOME=${tool 'jdk11'}", "PATH+MAVEN=${tool 'maven35'}/bin:${env.JAVA_HOME}/bin"]) {
                            checkout scm
                            sh 'mvn -B -V -U -e clean verify -Dsurefire.useFile=false -Dmaven.test.failure.ignore=true'
                            archiveArtifacts 'target/*.?ar'
                        }
                    } finally {
                        junit 'target/**/*.xml'  // Requires JUnit plugin
                    }
                }
            }

        }
    }
} catch (e) {
    node {
        mail to: "me@ne.me", subject: "Build failure - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
             body: "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>\n\n${e.stackTrace}"
    }
    throw e
} finally {
    node {
        if (currentBuild.result == 'UNSTABLE') {
            mail to: "me@ne.me", subject: "Build unstable - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                 body: "Unstable build: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>"
        } else if (currentBuild.result == null) { // null means success
            mail to: "me@ne.me", subject: "Build success - ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                 body: "Success of build: ${env.JOB_NAME} ${env.BUILD_NUMBER}\nSee results in Jenkins: <${env.BUILD_URL}>"
        }
    }
}
```
**Note:** Check your mail inbox. In case of build failure, the error stacktrace is also added to the email.

---

[← back to Lab 10](../10_failures.md)
