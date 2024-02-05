pipeline {
    agent any
    environment {
        APP_NAME = "react-demo-ci"
    }
    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }
        stage("Checkout from SCM") {
            steps {
                git branch: "main", credentialsId: "GitHub-Token", url: "https://github.com/IgnaLog/react-demo-gitops"
            }
        }
        stage("Update the Deployment Tags") {
            steps {
                sh """
                    cat manifiests/k8s/deployment.yaml
                    sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' manifiests/k8s/deployment.yaml
                    cat manifiests/k8s/deployment.yaml
                """
            }
        }
        stage("Push the changed deployment file to GitHub") {
            steps {
                sh """
                    git config --global user.name "IgnaLog"
                    git config --global user.email "ignacio.coding@gmail.com"
                    git add manifiests/k8s/deployment.yaml
                    git commit -m "Updated Deployment Manifest"
                """
                withCredentials([gitUsernamePassword(credentialsId: "GitHub-Token", gitToolName: "Default")]) {
                    sh "git push https://github.com/IgnaLog/react-demo-gitops main"
                }
            }
        }
    }
}