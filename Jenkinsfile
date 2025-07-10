// Jenkinsfile (Declarative Pipeline)

pipeline {
    agent any // Or specify a label for your Jenkins agent if you have specific agents (e.g., agent { label 'docker-build' })

    // Define environment variables
    environment {
        // --- Azure Container Registry (ACR) Variables ---
        // Replace with your ACR login server (e.g., myacr.azurecr.io)
        ACR_LOGIN_SERVER = 'atddockerreg.azurecr.io'
        // Replace with your ACR repository name (e.g., my-app)
        ACR_REPO_NAME = 'apps/python-webapp'
        // Jenkins credential ID for ACR login (e.g., 'azure-service-principal' or 'acr-creds')
        // This credential should be type 'Username with password' where username is the Service Principal App ID
        // and password is the Service Principal client secret, OR a Docker Hub credential type.
        // For Azure Service Principal, you might pass the JSON output to `az login --service-principal`
        AZURE_CREDENTIAL_ID = 'AZURE_CREDENTIALS' // ID of your Jenkins credential for Azure

        // --- Kubernetes (AKS) Variables ---
        // Replace with your AKS cluster name
        AKS_CLUSTER_NAME = 'atd-aks-cluster'
        // Replace with your Azure resource group where AKS is located
        AKS_RESOURCE_GROUP = 'k8s-group'
        // Kubernetes namespace to deploy to
        K8S_NAMESPACE = 'default'
        // Path to your Kubernetes manifests relative to the repo root
        K8S_MANIFESTS_PATH = 'aks'

        // --- Application/Build Variables ---
        // Version tag for your Docker image (e.g., using BUILD_NUMBER, Git commit SHA)
        // This creates a unique tag for each build
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                // Adjust this if your GitHub repo is private and needs credentials
                // For public repos, 'scm' is often enough
                // For private repos, use: checkout scm scm.scm.credentialsId = 'your-github-credential-id'
                git branch: 'main', url: 'https://github.com/avinashtckm/azure-course.git'
            }
        }

        stage('Build Application') {
            agent {
                docker {
                    image 'python:3.9-slim-buster' // Use a Python image that includes pip
                    // You might need 'args '-u 0'' if permissions issues arise inside the container,
                    // but often not needed for simple pip installs.
                }
            }
            steps {
                script {
                    // --- Choose ONE of the following build steps based on your application type ---

                    // Example: Node.js application
                    // sh 'npm install'
                    // sh 'npm run build' // If you have a build script

                    // Example: Python application (with pip requirements.txt)
                     sh 'pip install -r aks/k8s/requirements.txt'
                    // sh 'python setup.py build' // If you have a setup.py

                    // Example: Java Maven application
                    // sh 'mvn clean package -DskipTests' // Skips tests for faster build in CI, run tests in a separate stage
                    // sh 'ls -R target/' // To check generated artifacts, e.g., JAR/WAR file

                    // Example: Generic (if no specific build step is needed before Docker)
                    echo "No specific application build step defined, assuming Docker handles it or it's static."
                }
            }
        }

stage('Docker Build and Push') {
            steps {
                script {
                    withCredentials([string(credentialsId: "${env.AZURE_CREDENTIAL_ID}", variable: 'AZURE_SP_JSON')]) {
                        // NEW APPROACH: Log in to Azure CLI using the Service Principal JSON directly
                        // This uses a 'here-document' (<<EOF ... EOF) which is very robust.
                        // Groovy will first interpolate '${AZURE_SP_JSON}' with the actual JSON string.
                        // The shell then receives the 'az login' command with the JSON provided via standard input.
                        sh """
                        az login --service-principal --output none <<EOF
                        ${AZURE_SP_JSON}
EOF

                        # Get ACR login credentials and login Docker daemon (this part remains the same)
                        az acr login --name ${ACR_LOGIN_SERVER}
                        """
                    }

                    // Build the Docker image
                    sh "docker build -t ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:${IMAGE_TAG} ."

                    // Push the Docker image to ACR
                    sh "docker push ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:${IMAGE_TAG}"

                    // OPTIONAL: Tag with 'latest' and push (be careful with 'latest' in production)
                    // sh "docker tag ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:${IMAGE_TAG} ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:latest"
                    // sh "docker push ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:latest"
                }
            }
        }

stage('Deploy to AKS') {
            steps {
                script {
                    withCredentials([string(credentialsId: "${env.AZURE_CREDENTIAL_ID}", variable: 'AZURE_SP_JSON')]) {
                        // NEW APPROACH: Log in to Azure CLI using the Service Principal JSON directly
                        sh """
                        az login --service-service-principal --output none <<EOF
                        ${AZURE_SP_JSON}
EOF

                        # Get AKS credentials (this part remains the same)
                        az aks get-credentials --resource-group ${AKS_RESOURCE_GROUP} --name ${AKS_CLUSTER_NAME} --overwrite-existing
                        """
                    }

                    // Replace image placeholder in Kubernetes manifests
                    sh """
                    # Create a temporary directory for manifests
                    mkdir -p deploy_manifests
                    cp ${K8S_MANIFESTS_PATH}/*.yaml deploy_manifests/

                    # Replace placeholder in deployment.yaml (adjust filename if different)
                    # This assumes your deployment.yaml has 'image: YOUR_ACR_IMAGE_PLACEHOLDER'
                    sed -i "s|YOUR_ACR_IMAGE_PLACEHOLDER|${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:${IMAGE_TAG}|g" deploy_manifests/deployment.yaml

                    # Apply Kubernetes manifests
                    kubectl apply -f deploy_manifests/ -n ${K8S_NAMESPACE}

                    # Clean up temporary manifests
                    rm -rf deploy_manifests
                    """

                    echo "Deployment to AKS complete! Image: ${ACR_LOGIN_SERVER}/${ACR_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }

        // Optional: Add a testing/validation stage after deployment
        // stage('Post-Deployment Tests') {
        //     steps {
        //         // Example: Run a simple curl test against your service's external IP
        //         // If your service has an external IP, you can get it:
        //         // def serviceIp = sh(script: "kubectl get service your-service-name -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'", returnStdout: true).trim()
        //         // sh "curl http://${serviceIp}"
        //         echo "Running post-deployment tests (if any)..."
        //     }
        // }
    }

    // Optional: Post-build actions (e.g., notifications)
    post {
        always {
            cleanWs() // Clean up the workspace after the build
            echo "Pipeline finished."
        }
        success {
            echo "Pipeline succeeded!"
            // mail to: 'your-email@example.com', subject: "Jenkins Pipeline Succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline failed!"
            // mail to: 'your-email@example.com', subject: "Jenkins Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
