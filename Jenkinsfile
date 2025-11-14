pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "schinah/pythonwebsite"
        ARCH = 'amd64'
        DOCKER_ACCESS_TOKEN = credentials('docker_hub_token')
        DOCKER_ACCOUNT = credentials('schinah')
        CLOUD_BUILDER_NAME = 'schinah'
        IMAGE_NAME = 'IMAGE'
    }
     stages {
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            environment {
                BUILDX_URL = sh (returnStdout: true, script: 'curl -s https://raw.githubusercontent.com/docker/actions-toolkit/main/.github/buildx-lab-releases.json | jq -r ".latest.assets[] | select(endswith(\\"linux-$ARCH\\"))"').trim()
            }
            steps {
                    sh 'mkdir -vp ~/.docker/cli-plugins/'
                    sh 'curl --silent -L --output ~/.docker/cli-plugins/docker-buildx $BUILDX_URL'
                    sh 'chmod a+x ~/.docker/cli-plugins/docker-buildx'
                    sh 'echo "$DOCKER_ACCESS_TOKEN" | docker login --username $DOCKER_ACCOUNT --password-stdin'
                    sh 'docker buildx create --use --driver cloud "${DOCKER_ACCOUNT}/${CLOUD_BUILDER_NAME}"'
                    // Cache-only build
                    sh 'docker buildx build --platform linux/amd64,linux/arm64 --tag "$IMAGE_NAME" --output type=cacheonly .'
                    // Build and push a multi-platform image
                    sh 'docker buildx build --platform linux/amd64,linux/arm64 --push --tag "$IMAGE_NAME" .'
                    script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'main'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'website-deployment.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}
