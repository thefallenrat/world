def REPO = 'world'
def PACKAGES = []

pipeline {
    agent any
    options {
        skipDefaultCheckout true
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    def scmVars = checkout scm

                    def commit = scmVars.GIT_COMMIT
                    def commit_previous = scmVars.GIT_PREVIOUS_COMMIT
                    def commit_previous_successful = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT

                    def commit_previous_merge = sh(returnStdout: true, script: 'git log --merges --oneline --format=format:%H -n 1').trim()

                    echo "commit: ${commit}"
                    echo "commit_previous: ${commit_previous}"
                    echo "commit_previous_successful: ${commit_previous_successful}"
                    echo "commit_previous_merge: ${commit_previous_merge}"

                    def isMove = 'false'
                    if ( BRANCH_NAME == 'master' ) {
                        if ( commit == commit_previous_merge ) {
                            isMove = 'true'
                        }
                    }
                    echo "isMove: ${isMove}"
                    writeFile file: 'isMove.txt', text: isMove
                }
            }
        }
        stage('Prepare') {
            steps {
                script {
                    def changeLogSets = currentBuild.changeSets
                    for (int i = 0; i < changeLogSets.size(); i++) {
                        def entries = changeLogSets[i].items
                        for (int j = 0; j < entries.length; j++) {
                            def entry = entries[j]
                            echo "${entry.commitId} by ${entry.author} on ${new Date(entry.timestamp)}: ${entry.msg}"
                            def files = new ArrayList(entry.affectedFiles)
                            for (int k = 0; k < files.size(); k++) {
                                def file = files[k]
                                echo "${file.path}"
                                if ( file.path.contains('PKGBUILD') ){
                                    def pkg = file.path.minus('/PKGBUILD')
                                    PACKAGES << pkg
                                }
                            }
                        }
                    }
                    echo "PACKAGES: ${PACKAGES}"
                }
            }
        }
        stage('Stable') {
            environment {
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                branch 'master'
                expression { return readFile('isMove.txt').contains('false') }
            }
            steps {
                echo "PACKAGES: ${PACKAGES}"
                script {
                    for (pkg in PACKAGES) {
                        sh "buildpkg -p ${pkg} -z ${REPO}"
                    }
                }
            }
            post {
                success {
                    script {
                        for (pkg in PACKAGES) {
                            sh "deploypkg -x -p ${pkg} -r ${REPO}"
                        }
                    }
                }
            }
        }
        stage('Testing') {
            environment {
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                anyOf {
                    branch 'testing'
                    branch 'PR*'
                }
                expression { return readFile('isMove.txt').contains('false') }
            }
            steps {
                echo "PACKAGES: ${PACKAGES}"
                script {
                    for (pkg in PACKAGES) {
                        sh "buildpkg-testing -p ${pkg} -z ${REPO}-testing"
                    }
                }
            }
            post {
                success {
                    script {
                        for (pkg in PACKAGES) {
                            sh "deploypkg -x -p ${pkg} -r ${REPO}-testing"
                        }
                    }
                }
            }
        }
        stage('Move') {
            when {
                branch 'master'
                expression { return readFile('isMove.txt').contains('true') }
            }
            steps {
                sh "deploypkg -m -r ${REPO}-testing -t ${REPO}"
            }
        }
    }
}
