pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 20, unit: 'MINUTES')
    }

    parameters {
        booleanParam(
            name: 'DEPLOY_TO_PRODUCTION',
            defaultValue: true,
            description: 'Обновить production-сервер после успешной сборки'
        )
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    environment {
        DOCKER_IMAGE = 'samvelll/ruscar-bot'
        PROD_PATH = '/home/ubuntu/ruscar-bot'
        PROD_SSH_USER = 'ubuntu'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                script {
                    env.SHORT_COMMIT = sh(
                        script: 'git rev-parse --short=8 HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.SHORT_COMMIT}"
                    env.FULL_IMAGE = "${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
                }
                echo "Docker image: ${env.FULL_IMAGE}"
            }
        }

        stage('Validate Deployment Access') {
            when {
                expression { params.DEPLOY_TO_PRODUCTION }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: 'ruscar-production-ssh',
                        variable: 'SSH_KEY'
                    ),
                    string(
                        credentialsId: 'ruscar-production-host',
                        variable: 'PROD_HOST'
                    )
                ]) {
                    sh '''
                        chmod 600 "$SSH_KEY"
                        if ! ssh-keygen -y -f "$SSH_KEY" >/dev/null 2>&1; then
                            echo "Uploaded Jenkins SSH secret file is invalid"
                            exit 1
                        fi

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=accept-new \
                            "$PROD_SSH_USER@$PROD_HOST" \
                            "test -d '$PROD_PATH'"
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'docker build --pull -t "$FULL_IMAGE" .'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run --rm --entrypoint python "$FULL_IMAGE" -m py_compile /app/bot.py'
            }
        }

        stage('Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" --password-stdin
                        docker push "$FULL_IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression { params.DEPLOY_TO_PRODUCTION }
            }
            steps {
                withCredentials([
                    file(
                        credentialsId: 'ruscar-production-ssh',
                        variable: 'SSH_KEY'
                    ),
                    string(
                        credentialsId: 'ruscar-production-host',
                        variable: 'PROD_HOST'
                    )
                ]) {
                    sh '''
                        chmod 600 "$SSH_KEY"
                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=accept-new \
                            "$PROD_SSH_USER@$PROD_HOST" \
                            bash -s -- "$PROD_PATH" "$FULL_IMAGE" <<'REMOTE'
                        set -eu
                        deploy_path="$1"
                        image="$2"

                        cd "$deploy_path"
                        previous_image="$(sed -n 's/^RUSCAR_IMAGE=//p' .env | head -n 1)"
                        image_updated=false

                        rollback_on_failure() {
                            result=$?
                            trap - EXIT

                            if [ "$result" -ne 0 ] && [ "$image_updated" = "true" ] && [ -n "$previous_image" ]; then
                                echo "Deployment failed. Rolling back to $previous_image"
                                set +e
                                sed -i "s|^RUSCAR_IMAGE=.*|RUSCAR_IMAGE=$previous_image|" .env
                                sudo docker compose -f compose.server.yaml pull
                                sudo docker compose -f compose.server.yaml up -d --force-recreate
                                sleep 10
                                sudo docker compose -f compose.server.yaml ps
                            fi

                            exit "$result"
                        }

                        trap rollback_on_failure EXIT
                        sed -i "s|^RUSCAR_IMAGE=.*|RUSCAR_IMAGE=$image|" .env
                        image_updated=true
                        sudo docker compose -f compose.server.yaml pull
                        sudo docker compose -f compose.server.yaml up -d --force-recreate
                        sleep 10

                        container_status="$(sudo docker inspect -f '{{.State.Status}}' ruscar-bot)"
                        restart_count="$(sudo docker inspect -f '{{.RestartCount}}' ruscar-bot)"
                        if [ "$container_status" != "running" ] || [ "$restart_count" -ne 0 ]; then
                            sudo docker logs --tail=100 ruscar-bot
                            exit 1
                        fi

                        sudo docker compose -f compose.server.yaml ps
                        sudo docker logs --tail=30 ruscar-bot
                        trap - EXIT
                        REMOTE
                    '''.stripIndent()
                }
            }
        }
    }

    post {
        always {
            sh 'docker image rm "$FULL_IMAGE" >/dev/null 2>&1 || true'
        }
        success {
            echo "Deployment completed: ${env.FULL_IMAGE}"
        }
        failure {
            echo 'Pipeline failed. Check the failed stage and console log.'
        }
    }
}
