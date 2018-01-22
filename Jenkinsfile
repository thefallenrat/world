def REPO = 'world'
def PACKAGES = []
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

                    def current_commit = sh(returnStdout: true, script: 'git rev-parse @').trim()
                    def last_merge_commit = sh(returnStdout: true, script: 'git log --merges --oneline --format=format:%H -n 1').trim()

                    echo "current_commit: ${current_commit}"
                    echo "last_merge_commit: ${last_merge_commit}"

                    if ( BRANCH_NAME == 'master' ) {
                        if ( current_commit == last_merge_commit ) {
                            IS_MOVE = 'true'
                        }
                    } else if ( BRANCH_NAME == 'testing' ) {
                        if ( current_commit == last_merge_commit ) {
                            current_commit = sh(returnStdout: true, script: "git rev-parse @~").trim()
                        }
                    }
                    echo "IS_MOVE: ${IS_MOVE}"
                    
                    def changed_files = sh(returnStdout: true, script: "git show --pretty=format: --name-only ${current_commit}").tokenize()
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
        stage('Build') {
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
                }
            }
        }
    }
}
