pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {

        stage('Checkout') {
            steps {
                sh 'echo "Starting build process..."'
            }
        }

       stage('Build and Test') {
    steps {
        sh '''
            apk add --no-cache openjdk17-jre

            cd node-app
            npm ci
            npm test
        '''
    }
}

      stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonarqube') {
            sh '''
                apk add --no-cache openjdk17-jre

                export JAVA_HOME="/usr/lib/jvm/java-17-openjdk"
                export PATH="$JAVA_HOME/bin:$PATH"

                java -version

                cd node-app

                npm install --no-save sonar-scanner

                ./node_modules/.bin/sonar-scanner \
                  -Dsonar.projectKey=node-express-app \
                  -Dsonar.projectName="Node Express App" \
                  -Dsonar.sources=. \
                  -Dsonar.exclusions="node_modules/**,coverage/**" \
                  -Dsonar.host.url="$SONAR_HOST_URL"
            '''
        }
    }
}
       stage('Build and Push Docker Image') {
    environment {
        DOCKER_IMAGE = "zree7/node-js-app:${BUILD_NUMBER}"
    }

    steps {
        script {
            sh '''
                apk add --no-cache docker-cli

                docker --version

                docker build -t ${DOCKER_IMAGE} node-app
            '''

            def dockerImage = docker.image("${DOCKER_IMAGE}")

            docker.withRegistry(
                'https://index.docker.io/v1/',
                'docker-cred'
            ) {
                dockerImage.push()
                dockerImage.push("latest")
            }
        }
    }
}
        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "node-js-app-pipeline"
                GIT_USER_NAME = "Zree77"
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -rf repo-temp

                        git clone https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git repo-temp

                        cd repo-temp

                        git config user.email "zreeprojects@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        sed -i "s|image: .*|image: zree7/node-js-app:${BUILD_NUMBER}|g" node-app-manifests/deployment.yml

                        git add node-app-manifests/deployment.yml

                        git commit \
                            -m "Update Node.js image tag to ${BUILD_NUMBER} [skip ci]" \
                            || echo "No changes to commit"

                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                chown -R 105:109 "$WORKSPACE/node-app" || true
            '''
        }
    }
}
