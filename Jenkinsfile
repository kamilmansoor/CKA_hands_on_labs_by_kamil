/*
    Pipeline Author: KAMIL MANSOOR - SRE and DevOps
    Date: Febraury 2025
    
    Description: 
    This Jenkins pipeline automates the process of building, pushing, and deploying 
    Docker images to an Amazon EKS cluster. It supports test and production environments for now.

    Stages of Pipeline: 
    - The pipeline builds a Docker image using specific dockerfile specified for the environments.
    - Pushes the image to a private Docker Hub registry.
    - Updates the corresponding Kubernetes deployment in EKS.
    - Uses environment variables for dynamic configuration.

    Key Features of Pipeline:  
    - For test environment: It will trigger upon push in test branch only
    - For production environment: It will trigger upon tag push only in the repository
*/


pipeline {
    agent any

    environment {
        DOCKER_BUILDKIT = '1'
        DOCKER_REPO = 'hartech/test-repo'  // Default Docker repository
        DOCKER_REGISTRY = 'docker.io'
        MEM_LIMIT = '2g'
        CPU = '1'
        DOCKER_CREDENTIALS_NAME = 'dockercreds'
        NAMESPACE = 'test-namespace'
        TEST_DEPLOYMENT = 'deployment-test'
        PROD_DEPLOYMENT = 'deployment-prod'
        TEST_DEPLOYMENT_CONTAINER = '************'
        PROD_DEPLOYMENT_CONTAINER = '***********'

    }

    stages {
        stage('Print Pipeline Name and Repo') {
            steps {
                script {
                    echo "The pipeline is running for Environment: ${env.JOB_BASE_NAME} and the repo is ${env.DOCKER_REPO}"
                }
            }
        }

        stage('Setting up Image Tag and Image Name') {
            steps {
                script {

                    // Set image tag and dockerfile path based on environment, for test environment it will just attach latest as tag
                    if (env.JOB_BASE_NAME == 'test') {
                        image_tag = "test-latest"
                        dockerfile = 'dockerfiles/Dockerfile-test'

                    // Set image tag and dockerfile path based on environment, for prod environment it will fetch the latest tag from repo    
                    } else if (env.JOB_BASE_NAME == 'prod') {
                        def PROD_TAG = env.GIT_BRANCH?.replaceAll('^refs/tags/', '') ?: 'latest'
                        image_tag = "prod-${PROD_TAG}"
                        dockerfile = 'dockerfiles/Dockerfile'
                    }

                    // Set IMAGE_NAME and DOCKERFILE in the global environment
                    env.IMAGE_NAME = "${DOCKER_REPO}:${image_tag}" // Set dynamic IMAGE_NAME
                    env.DOCKERFILE = dockerfile // Set dynamic DOCKERFILE

                    echo "********** B U I L D    D E T A I L S **************"
                    echo "****************************************************"

                    echo "The environment is ${env.JOB_BASE_NAME}"
                    echo "Image Tag is set to: ${image_tag}"
                    echo "We are using the Dockerfile: ${DOCKERFILE}"
                    echo "The Final Image Name is set as ${env.IMAGE_NAME}"

                    echo "****************************************************"
                    echo "****************************************************"
                }
            }
        }

        stage('Building and Pushing Docker Image') {
            steps {
                script {
                    echo "Building Docker Image: ${env.IMAGE_NAME} using Dockerfile: ${DOCKERFILE}"

                    // Docker login
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_NAME, 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASSWORD')]) {
                        def loginStatus = sh(script: """
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USER" --password-stdin $DOCKER_REGISTRY
                        """, returnStatus: true)

                        if (loginStatus == 0) {
                            echo "✅ Docker login successful."
                        } else {
                            error "❌ Docker login failed."
                        }
                    }

                    // Docker build
                    def buildStatus = sh(script: """
                        docker buildx build --load --progress=plain --build-arg BUILDKIT_MEM_LIMIT=${env.MEM_LIMIT} --build-arg BUILDKIT_CPU_LIMIT=${env.CPU} -f ${env.DOCKERFILE} -t ${env.IMAGE_NAME} . --no-cache
                    """, returnStatus: true)

                    if (buildStatus == 0) {
                        echo "✅ Docker image build successful."
                    } else {
                        error "❌ Docker image build failed."
                    }

                    // Docker push
                    echo "************** Pushing Docker Image to Private Registry *************************"
                    def pushStatus = sh(script: "docker push ${env.IMAGE_NAME}", returnStatus: true)

                    if (pushStatus == 0) {
                        echo "✅ Docker image pushed successfully."
                    } else {
                        error "❌ Docker push failed."
                    }

                    echo "********************************************************************************"
                }
            }
        }

        stage('Apply manifest') {
            steps {
                script {
                    if (env.JOB_BASE_NAME == 'test') {
                        withCredentials([file(credentialsId: 'ekscreds-cl01', variable: 'KUBECONFIG')]) {
                            echo "📌 Image used for test deployment: ${env.IMAGE_NAME}"

                            // Check if the deployment exists before updating
                            def deployCheck = sh(script: "kubectl get deploy ${env.TEST_DEPLOYMENT} -n ${env.NAMESPACE}", returnStatus: true)

                            if (deployCheck == 0) {
                                echo "✅ Deployment exists. Proceeding with image update..."
                            } else {
                                error "❌ Deployment ${env.TEST_DEPLOYMENT} not found in namespace ${env.NAMESPACE}"
                            }

                            // Update the image
                            def updateStatus = sh(script: """
                                kubectl set image deploy/${env.TEST_DEPLOYMENT} ${env.TEST_DEPLOYMENT_CONTAINER}=${env.IMAGE_NAME} -n ${env.NAMESPACE}
                            """, returnStatus: true)

                            if (updateStatus == 0) {
                                echo "✅ Successfully updated deployment image to ${env.IMAGE_NAME}"
                            } else {
                                error "❌ Failed to update deployment image"
                            }

                            // Restart the deployment
                            def restartStatus = sh(script: "kubectl rollout restart deployment ${env.TEST_DEPLOYMENT} -n ${env.NAMESPACE}", returnStatus: true)

                            if (restartStatus == 0) {
                                echo "✅ Successfully restarted deployment"
                            } else {
                                error "❌ Deployment restart failed"
                            }
                        }
                    }

                    if (env.JOB_BASE_NAME == 'prod') {
                        withCredentials([file(credentialsId: 'ekscreds-cl01', variable: 'KUBECONFIG')]) {
                            echo "📌 Image used for production deployment: ${env.IMAGE_NAME}"

                            // Check if the deployment exists before updating
                            def deployCheck = sh(script: "kubectl get deploy ${env.PROD_DEPLOYMENT} -n ${env.NAMESPACE}", returnStatus: true)

                            if (deployCheck == 0) {
                                echo "✅ Deployment exists. Proceeding with image update..."
                            } else {
                                error "❌ Deployment ${env.PROD_DEPLOYMENT} not found in namespace ${env.NAMESPACE}"
                            }

                            // Update the image
                            def updateStatus = sh(script: """
                                kubectl set image deploy/${env.PROD_DEPLOYMENT} ${env.PROD_DEPLOYMENT_CONTAINER}=${env.IMAGE_NAME} -n ${env.NAMESPACE}
                            """, returnStatus: true)

                            if (updateStatus == 0) {
                                echo "✅ Successfully updated deployment image to ${env.IMAGE_NAME}"
                            } else {
                                error "❌ Failed to update deployment image"
                            }

                            // Restart the deployment
                            def restartStatus = sh(script: "kubectl rollout restart deployment ${env.PROD_DEPLOYMENT} -n ${env.NAMESPACE}", returnStatus: true)

                            if (restartStatus == 0) {
                                echo "✅ Successfully restarted deployment"
                            } else {
                                error "❌ Deployment restart failed"
                            }
                        }
                    }
                }
            }
        }
    }
}
