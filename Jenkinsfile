def REPO = 'world'
def PACKAGES = []
def GIT_CURRENT_COMMIT = ''
def GIT_LAST_MERGE_COMMIT = ''
def IS_MOVE = 'false'

pipeline {
    agent any
    options {
        skipDefaultCheckout()
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                    GIT_CURRENT_COMMIT = sh(returnStdout: true, script: 'git rev-parse @').trim()
                    GIT_LAST_MERGE_COMMIT = sh(returnStdout: true, script: 'git log --merges --oneline --format=format:%H -n 1').trim()

                    echo "GIT_CURRENT_COMMIT: ${GIT_CURRENT_COMMIT}"
                    echo "GIT_LAST_MERGE_COMMIT: ${GIT_LAST_MERGE_COMMIT}"

                    if ( BRANCH_NAME == 'master' ) {
                        if ( GIT_CURRENT_COMMIT == GIT_LAST_MERGE_COMMIT ) {
                            IS_MOVE = 'true'
                        }
                    } else if ( BRANCH_NAME == 'testing' ) {
                        if ( GIT_CURRENT_COMMIT == GIT_LAST_MERGE_COMMIT ) {
                            GIT_CURRENT_COMMIT = sh(returnStdout: true, script: "git rev-parse @~").trim()
                        }
                    }
                    echo "IS_MOVE: ${IS_MOVE}"
                }
            }
        }
        stage('Prepare') {
            when {
                expression { return  IS_MOVE == 'false' }
            }
            steps {
                script {
                    def changed_files = sh(returnStdout: true, script: "git show --pretty=format: --name-only ${GIT_CURRENT_COMMIT}").tokenize()
                    for (int i = 0; i < changed_files.size(); i++) {
                        def cfile = changed_files[i]
                        echo "Changed: " + cfile
                        if ( cfile.contains('PKGBUILD') ){
                            def pkg = cfile.minus('/PKGBUILD')
                            PACKAGES << pkg
                        }
                    }
                    echo "PACKAGES: ${PACKAGES}"
                }
            }
        }
        stage('Move') {
            when {
                branch 'master'
                expression { return  IS_MOVE == 'true' }
            }
            steps {
                sh "deploypkg -m -r ${REPO}-testing -t ${REPO}"
            }
        }
        stage('Stable') {
            environment {
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                branch 'master'
                expression { return  IS_MOVE == 'false' }
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
                    deleteDir()
                }
                failure {
                    deleteDir()
                }
                aborted {
                    deleteDir()
                }
            }
        }
        stage('Testing') {
            environment {
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                branch 'testing'
                expression { return  IS_MOVE == 'false' }
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
                    deleteDir()
                }
                failure {
                    deleteDir()
                }
                aborted {
                    deleteDir()
                }
            }
        }
    }
}
