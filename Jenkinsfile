pipeline {
    agent any
    environment{
        REPO = 'world'
        CMD = 'buildpkg'
    }
    options {
        skipDefaultCheckout false
    }
    stages {
        stage('Prepare') {
            environment {
                COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                LAST_MERGE = sh(returnStdout: true, script: 'git log --merges --oneline --format=format:%H -n 1').trim()
            }
            steps {
                sh '''
                    isMove=false
                    if [[ ${BRANCH_NAME} == 'master' ]];then
                        [[ ${COMMIT} == ${LAST_MERGE} ]] && isMove=true
                    fi
                    echo ${isMove} > isMove.txt
                    packages=()
                    gitChangeset=$(git show --pretty=format: --name-only HEAD)
                    for f in ${gitChangeset[@]};do
                        if [[ "$f" == */PKGBUILD ]];then
                            pkg=${f%/PKGBUILD}
                            packages+=(${pkg})
                        fi
                    done
                    echo ${packages[*]} > packages.txt
                '''
            }
        }
        stage('BuildInStable') {
            environment {
                BUILDPKG = "${CMD}"
                REPO_DEST = "${REPO}"
                PACKAGES = readFile('packages.txt')
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                branch 'master'
                expression { return readFile('isMove.txt').contains('false') }
            }
            steps {
                sh '''
                    for pkg in ${PACKAGES[@]};do
                        ${BUILDPKG} -p ${pkg} -z ${REPO_DEST}
                    done
                '''
            }
            post {
                success {
                    sh '''
                        for pkg in ${PACKAGES[@]};do
                            deploypkg -x -p ${pkg} -r ${REPO_DEST}
                        done
                    '''
                    archiveArtifacts '**/*.log'
                }
            }
        }
        stage('BuildInTesting') {
            environment {
                BUILDPKG = "${CMD}-${BRANCH_NAME}"
                REPO_DEST = "${REPO}-${BRANCH_NAME}"
                PACKAGES = readFile('packages.txt')
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
                branch 'testing'
                expression { return readFile('isMove.txt').contains('false') }
            }
            steps {
                sh '''
                    for pkg in ${PACKAGES[@]};do
                        ${BUILDPKG} -p ${pkg} -z ${REPO_DEST}
                    done
                '''
            }
            post {
                success {
                    sh '''
                        for pkg in ${PACKAGES[@]};do
                            deploypkg -x -p ${pkg} -r ${REPO_DEST}
                        done
                    '''
                    archiveArtifacts '**/*.log'
                }
            }
        }
        stage('MoveInStable') {
            environment {
                REPO_DEST = "${REPO}"
                REPO_SRC = "${REPO}-testing"
            }
            when {
                branch 'master'
                expression { return readFile('isMove.txt').contains('true') }
            }
            steps {
                sh '''
                    deploypkg -r ${REPO_SRC} -t ${REPO_DEST} -m
                '''
            }
        }
    }
}
