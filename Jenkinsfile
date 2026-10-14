
pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        IMAGE_REPO = 'kunalparihar8/jenkins-demo-web'
    }

    stages {
        stage('Build and Push Docker Image') {
            agent {
                label 'docker-worker'
            }

            steps {
                checkout scm

                script {
                    echo "Building ${IMAGE_REPO}:${BUILD_NUMBER}"
                }

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "$DOCKERHUB_TOKEN" |
                          docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker build \
                          -t "$IMAGE_REPO:$BUILD_NUMBER" \
                          -t "$IMAGE_REPO:latest" .

                        docker push "$IMAGE_REPO:$BUILD_NUMBER"
                        docker push "$IMAGE_REPO:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy using Ansible') {
            agent {
                label 'ansible-node'
            }

            steps {
                checkout scm

                sshagent(credentials: ['ssh-managed-host']) {
                    sh '''
                        set -eu

                        export ANSIBLE_HOST_KEY_CHECKING=False

                        ansible -i inventory.ini managed \
                          -m ping

                        ansible-playbook \
                          -i inventory.ini \
                          deploy.yml \
                          --extra-vars \
                          "image_name=${IMAGE_REPO} image_tag=${BUILD_NUMBER}"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            agent {
                label 'ansible-node'
            }

            steps {
                sshagent(credentials: ['ssh-managed-host']) {
                    sh '''
                        set -eu

                        ssh -o StrictHostKeyChecking=no \
                          ec2-user@3.110.175.65 \
                          "docker ps --filter name=jenkins-demo-web"

                        curl --fail \
                          http://3.110.175.65:8080/
                    '''
                }
            }
        }
    }
}
