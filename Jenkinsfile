pipeline {
    agent { label "Jenkins-Agent" }

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Docker Image Tag'
        )
    }

    environment {
        APP_NAME = "register-app-pipeline"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', 
                credentialsId: 'GitHub', 
                url: 'https://github.com/RAM12837/register-app-gitops'
            }
        }

        stage("Update the Deployment Tags") {
            steps {
                sh """
                    echo "received image tag is ${IMAGE_TAG}"
                    cat deployment.yaml
                    sed -i "s|image: remson001/register-app-pipeline:.*|image: remson001/register-app-pipeline:${IMAGE_TAG}|g" deployment.yaml                    echo "update deployment.yaml with new image tag ${IMAGE_TAG}"
                    cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                   git config --global user.name "RAM12837"
                   git config --global user.email "ramkumardamde1432@gmail.com"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'GitHub', gitToolName: 'Default')]) {
                  sh """
                    git add deployment.yaml
                    git commit -m "Updated the deployment.yaml with new image tag ${IMAGE_TAG}"
                    git push origin main
                  """
                }
            }
        }
      
    }
}
