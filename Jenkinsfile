pipeline {
    agent any
    stages {
        stage('Prepare') {
            steps {
                sh '''
                    gitCommit=$(git rev-parse HEAD)
                    gitChangeset=$(git show --pretty=format: --name-only ${gitCommit})
                    gitLastMerge=$(git log --merges --oneline --format=format:%H -n 1)
                    repo_dest='world'
                    repo_src='none'
                    isMove=false
                    case ${BRANCH_NAME} in
                        'testing')
                            repo_dest=${repo_dest}-${BRANCH_NAME}
                            cmd="buildpkg-${BRANCH_NAME}"
                        ;;
                        'master')
                            cmd="buildpkg"
                            if [[ ${gitCommit} == ${gitLastMerge} ]];then
                                isMove=true
                                repo_src=${repo_dest}-testing
                            fi
                        ;;
                        PR-*)
                            repo_dest=${repo_dest}-testing
                            cmd="buildpkg-testing"
                        ;;
                    esac
                    echo ${isMove} > isMove.txt
                    echo ${cmd} > cmd.txt
                    echo ${repo_dest} > repo_dest.txt
                    echo ${repo_src} > repo_src.txt
                    packages=()
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
        stage('Build') {
            environment {
                BUILDPKG = readFile('cmd.txt')
                REPO_DEST = readFile('repo_dest.txt')
                PACKAGES = readFile('packages.txt')
                BUILDBOT_GPGP = credentials('BUILDBOT_GPGP')
            }
            when {
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
                }
            }
        }
        stage('Move') {
            environment {
                REPO_DEST = readFile('repo_dest.txt')
                REPO_SRC = readFile('repo_src.txt')
            }
            when {
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
