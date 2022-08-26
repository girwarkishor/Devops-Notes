# DevSecOps

## Prerequisite

1. Setup a Jenkins Server on any cloud provider

The below pipeline is builded using declarative approach. In this approach while writing the groovy script I found some important points

1. In declarative approach the script starts with **pipeline**
2. Without **agent** you cannot run the pipeline
3. **environment** block is used for defining the variables
4. **stages** block contains **stage** block which is used to specify the current stage
5. **steps** are the main block of every **stage**
6. Use **Pipeline Syntax** link to generate the groovy scripts
7. For running SonarQube scanner you have to install SonarQube scanner plugin
8. You have to do some configurations to run SonarQube. See ScreenShots
9. You need to install Node Js plugin and do the configuration
10. For showing the audit report on html file you need to install **HTMLPublish** plugins.
11. After that you need to generate the groovy script for it. Search **publishHTML: Publish HTML** reports on Sample Step dropdown and follow the steps

```groovy
pipeline {
    agent any
    environment{
        imageName = "girwar/nodejs-app"
        secret = credentials("repo-secret")
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/girwarkishor/node-app.git'
            }
        }
        stage('Code Scanning'){
            environment{
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                    sh '${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=node-microsvc -Dsonar.sources=. -Dsonar.host.url=http://20.245.233.156:9000 -Dsonar.login=sqp_a4ba432784c3bf206667cacc1cdf1020f0d706ef'
            }
        }
        stage('Build'){
            steps{
                nodejs("nodejs"){
                    sh 'npm install'
                }
            }
        }
        stage('Dependency Scanning'){
            steps{
                nodejs("nodejs"){
                    sh '''npm audit fix || true
                    npm audit  --audit-level=none
                    npm install -g npm-audit-html
                    npm audit --json | npm-audit-html'''
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'npm-audit.html', reportName: 'Rcov Report', reportTitles: ''])
                }
            }
        }
        stage('Docker build and publish'){
            steps{
                sh '''docker build -t $imageName:$BUILD_ID .
                docker build -t $imageName:latest .
                export DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=$secret
                docker trust sign $imageName:$BUILD_ID
                docker trust sign $imageName:latest'''
            }
        }
        stage("Scan Image"){
            steps{
                sh 'clair-scanner -r report.html --ip 172.17.0.1 $imageName:$BUILD_ID || exit 0'
            }
        }
        stage("Deploy"){
            steps{
                sh '''cd resource-manifests
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                sleep 30
                IPADDRESS=`kubectl get services nodejs-app-lb --output jsonpath=\'{.status.loadBalancer.ingress[0].ip}\'`
                '''
            }
        }
    }
}
```

```

```
