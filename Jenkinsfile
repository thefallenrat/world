node {
    stage('Checkout') {
        checkout scm
    }
    stage('Build') {
        sh "/var/lib/jenkins/build.sh"
    }
    stage('Repo') {
        sh "/var/lib/jenkins/repo.sh"
    }
}
