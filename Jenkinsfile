

pipeline {
    agent any

    environment {
        DEPLOY_DIR = 'C:\\inetpub\\wwwroot\\Django_app'
        PYTHON = 'C:\\Users\\Administrator\\AppData\\Local\\Programs\\Python\\Python310\\python.exe'
        APP_POOL = 'Django_app'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Syed-Haider-rizvi/Django_app.git'
            }
        }

        stage('Python Environment') {
            steps {
                powershell '''
                    if (!(Test-Path "manage.py")) {
                        throw "manage.py not found in repository root"
                    }

                    if (!(Test-Path ".venv")) {
                        & $env:PYTHON -m venv .venv
                    }

                    & ".venv\\Scripts\\python.exe" -m pip install --upgrade pip

                    if (Test-Path "requirements.txt") {
                        & ".venv\\Scripts\\python.exe" -m pip install -r requirements.txt
                    }

                    if ($LASTEXITCODE -ne 0) {
                        throw "Python dependency installation failed"
                    }
                '''
            }
        }

        stage('Django Check') {
            steps {
                powershell '''
                    & ".venv\\Scripts\\python.exe" manage.py check

                    if ($LASTEXITCODE -ne 0) {
                        throw "Django validation failed"
                    }
                '''
            }
        }

        stage('Deploy to IIS') {
            steps {
                powershell '''
                    New-Item -ItemType Directory -Force `
                        -Path $env:DEPLOY_DIR | Out-Null

                    robocopy $env:WORKSPACE $env:DEPLOY_DIR `
                        /E /XD .git .venv __pycache__ `
                        /XF Jenkinsfile .env

                    if ($LASTEXITCODE -ge 8) {
                        throw "Deployment file copy failed"
                    }

                    if (!(Test-Path "$env:DEPLOY_DIR\\web.config")) {
                        throw "IIS web.config missing from deployment"
                    }

                    if (!(Test-Path "$env:DEPLOY_DIR\\manage.py")) {
                        throw "manage.py missing from deployment"
                    }

                    if (!(Test-Path "$env:DEPLOY_DIR\\.venv")) {
                        & $env:PYTHON -m venv "$env:DEPLOY_DIR\\.venv"
                    }

                    $PY = "$env:DEPLOY_DIR\\.venv\\Scripts\\python.exe"

                    & $PY -m pip install --upgrade pip

                    & $PY -m pip install -r "$env:DEPLOY_DIR\\requirements.txt"

                    if ($LASTEXITCODE -ne 0) {
                        throw "Production dependencies failed"
                    }

                    & $PY -m pip install wfastcgi

                    if ($LASTEXITCODE -ne 0) {
                        throw "wfastcgi installation failed"
                    }
                '''
            }
        }

        stage('Database Migration') {
            steps {
                powershell '''
                    $PY = "$env:DEPLOY_DIR\\.venv\\Scripts\\python.exe"

                    & $PY "$env:DEPLOY_DIR\\manage.py" migrate --noinput

                    if ($LASTEXITCODE -ne 0) {
                        throw "Database migration failed"
                    }
                '''
            }
        }

        stage('Collect Static Files') {
            steps {
                powershell '''
                    $PY = "$env:DEPLOY_DIR\\.venv\\Scripts\\python.exe"

                    & $PY "$env:DEPLOY_DIR\\manage.py" collectstatic --noinput

                    if ($LASTEXITCODE -ne 0) {
                        throw "Static collection failed"
                    }
                '''
            }
        }

        stage('Restart IIS') {
            steps {
                powershell '''
                    & "$env:WINDIR\\System32\\inetsrv\\appcmd.exe" `
                        recycle apppool /apppool.name:$env:APP_POOL

                    if ($LASTEXITCODE -ne 0) {
                        throw "IIS App Pool restart failed"
                    }
                '''
            }
        }
    }

    post {
        success {
            echo 'Django application deployed successfully to IIS!'
        }

        failure {
            echo 'Django deployment failed. Check Jenkins Console Output.'
        }
    }
}
