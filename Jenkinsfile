pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "xicoria/echo"
        GIT_SHORT_COMMIT = "${env.GIT_COMMIT[0..6]}"
    }

    stages {
        stage('Install dependencies'){
             agent { 
                docker {
                    image 'node:current-alpine3.13'
                    reuseNode true
                }
            }
            steps {
                echo 'Installing dependencies'
                sh 'cd src;npm install;'
            }
        }
        stage('Run tests'){
             agent { 
                docker {
                    image 'node:current-alpine3.13'
                    reuseNode true
                }
            }
            steps {
                echo 'Running tests'
                sh 'cd src;npm test;'
            }
        }
        stage('Build docker image'){
            steps {
                echo 'Creating image'
                script {
                    app = docker.build(DOCKER_IMAGE_NAME, "-f docker/app/Dockerfile .")
                }
            }
        }
        stage('Push image to registry') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("$GIT_SHORT_COMMIT")
                    }
                }
            }
        }
        stage('Deploy'){
            steps {
                echo 'Deploying to kubernetes'
                kubernetesDeploy(
                    kubeconfigId: 'kube_config',
                    configs: 'kubernetes/app/manifest.yaml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}

