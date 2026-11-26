pipeline{
    agent {
        docker{
            label "linux"
            image "ghcr.io/astral-sh/uv:python3.14-bookworm-slim"
            args "-v /var/cache/ci/uv:/cache/uv"
        }
    }
    
    environment{
        HOME                  = '/tmp'
        XDG_CACHE_HOME        = '/cache'
        UV_CACHE_DIR          = '/cache/uv'
        UV_PYTHON_INSTALL_DIR = '/cache/uv-python'
    }

    stages{
        stage("Initialisation"){
            steps {
                echo "Blop"
                sh 'cat .python-version; python3 --version; uv python find'
                sh "uv --version"
                sh "uv cache dir"
                sh "uv sync --locked -v"
            }
        }
    }
}