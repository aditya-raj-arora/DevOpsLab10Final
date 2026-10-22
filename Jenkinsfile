pipeline {
    agent any

    environment {
        IMAGE     = 'adityarajarora123/traffic-system'   // dockerhub-user/repo
        TAG       = "${BUILD_NUMBER}"                    // unique tag per build: 1, 2, 3...
        APP       = 'traffic-system'
        TEST_PORT = '8092'
    }

    triggers {
        pollSCM('H/2 * * * *')      // check GitHub every ~2 minutes; a new commit starts a build
    }

    parameters {
        booleanParam(name: 'ROLLBACK', defaultValue: false, description: 'Run the extra test suite')
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/aditya-raj-arora/DevOpsLab10Final.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %IMAGE%:%TAG% -t %IMAGE%:latest ."
            }
        }

        stage('Test Application') {
            steps {
                bat "docker rm -f %APP%-test || exit 0"                          // remove an old test container if any
                bat "docker run -d --name %APP%-test -p %TEST_PORT%:80 %IMAGE%:%TAG%"
                sleep 3                                                          // give nginx time to start
                bat "curl -f --retry 5 --retry-delay 2 --retry-all-errors http://localhost:%TEST_PORT%"                       // -f: fail unless HTTP 200
                bat "docker rm -f %APP%-test"
            }
        }

        

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat "docker push %IMAGE%:%TAG%"
                    bat "docker push %IMAGE%:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    bat 'kubectl apply -f deployment.yaml'
                    bat "kubectl set image deployment/%APP% %APP%=%IMAGE%:%TAG%"
                    bat "kubectl rollout status deployment/%APP% --timeout=180s"
                }
            }
        }

        stage('Verify Deployment') {
            when {
                expression { params.ROLLBACK == false }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    bat "kubectl get deployment %APP%"
                    bat "kubectl get pods -l app=%APP% -o wide"
                    bat "kubectl get service %APP%-service"
                }
            }
        }
        stage('Verify or Roll Back') {
            when {
                expression { params.ROLLBACK == true }
            }
            steps {
                script {
                    // returnStatus: true -> give me the exit code instead of failing the build
                    int rc = bat(script: "kubectl rollout status deployment/%APP% --timeout=120s", returnStatus: true)
                    if (rc != 0) {
                        echo "Release ${env.TAG} is NOT healthy. Rolling back to the previous stable release."
                        bat(script: "kubectl get pods -l app=%APP% -o wide", returnStatus: true)
                        bat(script: "kubectl get events --field-selector type=Warning --sort-by=.lastTimestamp", returnStatus: true)
                        int undo = bat(script: "kubectl rollout undo deployment/%APP%", returnStatus: true)
                        if (undo == 0) {
                            bat "kubectl rollout status deployment/%APP% --timeout=120s"
                            bat "kubectl rollout history deployment/%APP%"
                        } else {
                            echo 'No earlier revision exists (first deployment), nothing to roll back to.'
                        }
                        error "Release ${env.TAG} failed its health check and was rolled back."
                    }
                }
                bat "kubectl get deployment,pods,service -l app=%APP% -o wide"
                bat "kubectl get service %APP%-service"
                bat "curl -f --retry 5 --retry-delay 2 --retry-all-errors http://localhost:%NODE_PORT%/health.html"
            }
        }
    }

    post {
        success { echo "Deployed ${IMAGE}:${TAG}. Open http://localhost:30100" }
        failure { echo 'Pipeline FAILED - check the stage logs above.' }
        always  { bat 'docker logout || exit 0' }
    }
}
