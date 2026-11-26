pipeline{
    agent {
        docker{
            label "linux"
            image "ghcr.io/astral-sh/uv:python3.14-bookworm-slim"
            args "-v /var/cache/ci/uv:/cache/uv"
        }
    }
    stages{
        stage("Initialisation"){
            steps {
                echo "Blop"
                sh "uv --version"
            }
        }
    }
}