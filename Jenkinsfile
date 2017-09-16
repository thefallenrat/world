pipeline {
    agent any
    stages {
        stage('Prepare') {
            steps {
                sh '''
                    GIT_COMMIT=$(git rev-parse HEAD)
                    GIT_COMMIT_FILES=$(git show --pretty=format: --name-only ${GIT_COMMIT})
                    echo "GIT_COMMIT_FILES: ${GIT_COMMIT_FILES[@]}"
                    REPO='world'
                    case ${BRANCH_NAME} in
                        'testing'|'staging')
                            REPO=${REPO}-${BRANCH_NAME}
                            BUILDPKG="buildpkg-${BRANCH_NAME}"
                        ;;
                        'master')
                            BUILDPKG="buildpkg"
                        ;;
                        PR-*)
                            REPO=${REPO}-testing
                            BUILDPKG="buildpkg-testing"
                        ;;
                    esac
                    echo ${BUILDPKG} > cmd.txt
                    echo ${REPO} > repo.txt
                    PACKAGE=none
                    for f in ${GIT_COMMIT_FILES[@]};do
                        echo "tracked changed file: $f"
                        if [[ "$f" == */PKGBUILD ]];then
                            PACKAGE=${f%/PKGBUILD}
                            echo ${PACKAGE}
                        fi
                    done
                    echo ${PACKAGE} > package.txt
                '''
            }
        }
        stage('Build') {
            environment {
                BUILDPKG = readFile('cmd.txt')
                REPO = readFile('repo.txt')
                PACKAGE = readFile('package.txt')
            }
            steps {
                sh '''
                    ARGS=()
                    case ${PACKAGE} in
                        llvm) ARGS+=(-a multilib) ;;
                    esac
                    ${BUILDPKG} -p ${PACKAGE} -z ${REPO} "${ARGS[@]}"
                '''
            }
            post {
                success {
                    sh '''
                        deploypkg -x -p ${PACKAGE} -r ${REPO}
                    '''
                }
            }
        }
    }
}
