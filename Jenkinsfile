#!/usr/bin/env groovy

def GIT_REMOTE = 'https://github.com/asequeira/argentesting.git' // Groovy style
def isRelease
def RETRIES = 3
def TIMEOUT = 10

node('mySlave') {

    isRelease = env.isRelease == "true"

    try {

        stage('checkout') {
            commit = env.Commit                                                                                             // env.Commit or param.Commit si a job argument
            checkout scm: [$class: 'GitSCM', branches: [[name: commit]], userRemoteConfigs: [[url: GIT_REMOTE]] ]           // Checking out the code
            currentBuild.displayName = '#' + commit
        }

        stage('build') {
            sh "mvn clean install"
        }

        stage('integration tests') {
            lock "QA environment"
            sh "./run-it-test.sh"
        }

        stage('deploy'){
            sh "mvn deploy:deploy-file -Dfile=target/artifact-${commit}.jar -DgroupId=com.medallia -DartifactId=test -Dversion=${commit}"
        }

        if (isRelease) {
            echo "Promoting to production"
            stage('promote') {
                def version = env.Version
                upload(version)
                tag(version)
            }
        }

    } catch (err) {
        notifyBuild('FAILURE')      // Calling a Groovy function
        throw err                   // Rethrowing so build fails
    }
}

def notifyBuild(String buildStatus) {
    emailext(
            subject: "Build failed!",
            body: "See ${env.BUILD_URL}",
            to: "name@mail.com"
    )
}

def tag(String version) {
    sh "git tag -a ${version} -m 'Release version ${version}'"
    sh "git push origin ${version}"
}

def upload(String version) {
    retry(RETRIES) {
        timeout(time: TIMEOUT, unit: 'SECONDS') {
            sh "aws s3 target/artifact-${version}.jar cp s3://medallia/releases/${version}/artifact-${version}.jar"
        }
    }
}
