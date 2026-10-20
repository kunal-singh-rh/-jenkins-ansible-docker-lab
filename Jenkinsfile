
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
                        credentialsId: 'docker-hub',
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

                sshagent(credentials: ['managed-host']) {
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
                sshagent(credentials: ['managed-host']) {
                    sh '''
                        set -eu

                        sudo ssh -o StrictHostKeyChecking=no \
                          ec2-user@13.203.198.100 \
                          "docker ps --filter name=jenkins-demo-web"

                        sudo curl --fail \
                          http://13.203.198.100:8080/
                    '''
                }
            }
        }
    }
}
