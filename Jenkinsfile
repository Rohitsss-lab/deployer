pipeline {
    agent any
    parameters {
        string(name: 'DEPLOY_VERSION', defaultValue: '', description: 'Umbrella version to deploy e.g. 1.0.0')
    }
    environment {
        GIT_REPO_URL = 'https://github.com/Rohitsss-lab/deployer.git'
        SERVER_IP    = '192.168.3.178'
        SERVER_USER  = 'root'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout Umbrella Tag') {
            steps {
                script {
                    if (!params.DEPLOY_VERSION?.trim()) {
                        error "DEPLOY_VERSION is required — enter umbrella version e.g. 1.0.0"
                    }
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "refs/tags/v${params.DEPLOY_VERSION}"]],
                        userRemoteConfigs: [[
                            url: "${env.GIT_REPO_URL}",
                            credentialsId: 'github-token'
                        ]]
                    ])
                    echo "Checked out deployer at tag v${params.DEPLOY_VERSION}"
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
'''
                    bat '"C:\\Program Files\\Python313\\python.exe" parse_versions.py'

                    env.FRONTEND_VERSION = readFile('FRONTEND_VERSION.txt').replaceAll('[^0-9.]', '').trim()
                    env.BACKEND_VERSION  = readFile('BACKEND_VERSION.txt').replaceAll('[^0-9.]', '').trim()

                    echo "Umbrella  : v${params.DEPLOY_VERSION}"
                    echo "frontend  : v${env.FRONTEND_VERSION}"
                    echo "backend   : v${env.BACKEND_VERSION}"
                    echo "Server    : ${env.SERVER_USER}@${env.SERVER_IP}"
                }
            }
        }
        stage('Deploy frontend') {
            steps {
                echo "Deploying frontend v${env.FRONTEND_VERSION} to ${env.SERVER_IP}"
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    bat """
                        icacls "%SSH_KEY%" /inheritance:r
                        icacls "%SSH_KEY%" /grant:r "SYSTEM:F"
                        icacls "%SSH_KEY%" /grant:r "Administrators:F"
                        ssh -o StrictHostKeyChecking=no -i "%SSH_KEY%" ${env.SERVER_USER}@${env.SERVER_IP} "cd /root/frontend && git fetch --tags && git checkout tags/v${env.FRONTEND_VERSION} -f && npm install --production && pm2 restart frontend || pm2 start src/index.js --name frontend && pm2 save"
                    """
                }
                echo "frontend v${env.FRONTEND_VERSION} is LIVE on ${env.SERVER_IP}:3001"
            }
        }
        stage('Deploy backend') {
            steps {
                echo "Deploying backend v${env.BACKEND_VERSION} to ${env.SERVER_IP}"
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    bat """
                        icacls "%SSH_KEY%" /inheritance:r
                        icacls "%SSH_KEY%" /grant:r "SYSTEM:F"
                        icacls "%SSH_KEY%" /grant:r "Administrators:F"
                        ssh -o StrictHostKeyChecking=no -i "%SSH_KEY%" ${env.SERVER_USER}@${env.SERVER_IP} "cd /root/backend && git fetch --tags && git checkout tags/v${env.BACKEND_VERSION} -f && npm install --production && pm2 restart backend || pm2 start src/index.js --name backend && pm2 save"
                    """
                }
                echo "backend v${env.BACKEND_VERSION} is LIVE on ${env.SERVER_IP}:3002"
            }
        }
    }
    post {
        success {
            echo "==========================================="
            echo "DEPLOY SUCCESS"
            echo "Umbrella  v${params.DEPLOY_VERSION}"
            echo "frontend  v${env.FRONTEND_VERSION} LIVE on port 3001"
            echo "backend   v${env.BACKEND_VERSION}  LIVE on port 3002"
            echo "Server    ${env.SERVER_IP}"
            echo "==========================================="
        }
        failure { echo "Deployment FAILED" }
    }
}
