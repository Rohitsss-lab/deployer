pipeline {
    agent any
    parameters {
        string(name: 'DEPLOY_VERSION', defaultValue: '', description: 'Umbrella version to deploy e.g. 1.0.5')
    }
    environment {
        GIT_REPO_URL = 'https://github.com/Rohitsss-lab/deployer.git'
        SERVER_IP    = 'YOUR_SERVER_IP'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout Umbrella Tag') {
            steps {
                script {
                    if (!params.DEPLOY_VERSION?.trim()) {
                        error "Enter DEPLOY_VERSION — e.g. 1.0.5"
                    }
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "refs/tags/v${params.DEPLOY_VERSION}"]],
                        userRemoteConfigs: [[
                            url: "${env.GIT_REPO_URL}",
                            credentialsId: 'github-token'
                        ]]
                    ])
                    echo "Checked out deployer at umbrella tag v${params.DEPLOY_VERSION}"
                }
            }
        }
        stage('Read Versions') {
            steps {
                script {
                    writeFile file: 'parse_versions.py', text: '''
import json

with open("versions.json", "r") as f:
    data = json.load(f)

frontend = data.get("frontend", "").strip()
backend  = data.get("backend",  "").strip()

with open("FRONTEND_VERSION.txt", "w", newline="") as f:
    f.write(frontend)

with open("BACKEND_VERSION.txt", "w", newline="") as f:
    f.write(backend)

print(f"frontend = {frontend}")
print(f"backend  = {backend}")
'''
                    bat '"C:\\Program Files\\Python313\\python.exe" parse_versions.py'

                    env.FRONTEND_VERSION = readFile('FRONTEND_VERSION.txt')
                                            .replaceAll('[^0-9.]', '').trim()
                    env.BACKEND_VERSION  = readFile('BACKEND_VERSION.txt')
                                            .replaceAll('[^0-9.]', '').trim()

                    echo "==========================================="
                    echo "Umbrella  : v${params.DEPLOY_VERSION}"
                    echo "frontend  : v${env.FRONTEND_VERSION}"
                    echo "backend   : v${env.BACKEND_VERSION}"
                    echo "==========================================="
                }
            }
        }
        stage('Deploy frontend') {
            steps {
                echo "Deploying frontend v${env.FRONTEND_VERSION}"
                sshagent(credentials: ['deploy-ssh-key']) {
                    bat """
                        ssh -o StrictHostKeyChecking=no root@${env.SERVER_IP} "cd /root/frontend && git fetch --tags && git checkout tags/v${env.FRONTEND_VERSION} -f && npm install --production && pm2 restart frontend || pm2 start src/index.js --name frontend && pm2 save"
                    """
                }
                echo "frontend v${env.FRONTEND_VERSION} is LIVE"
            }
        }
        stage('Deploy backend') {
            steps {
                echo "Deploying backend v${env.BACKEND_VERSION}"
                sshagent(credentials: ['deploy-ssh-key']) {
                    bat """
                        ssh -o StrictHostKeyChecking=no root@${env.SERVER_IP} "cd /root/backend && git fetch --tags && git checkout tags/v${env.BACKEND_VERSION} -f && npm install --production && pm2 restart backend || pm2 start src/index.js --name backend && pm2 save"
                    """
                }
                echo "backend v${env.BACKEND_VERSION} is LIVE"
            }
        }
    }
    post {
        success {
            echo "==========================================="
            echo "DEPLOY SUCCESS"
            echo "Umbrella  v${params.DEPLOY_VERSION}"
            echo "frontend  v${env.FRONTEND_VERSION} LIVE"
            echo "backend   v${env.BACKEND_VERSION}  LIVE"
            echo "==========================================="
        }
        failure { echo "Deployment FAILED" }
    }
}
