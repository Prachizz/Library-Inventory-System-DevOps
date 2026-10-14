pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'orange18036/team3-library'
        DOCKER_IMAGE_GHCR = 'ghcr.io/orange18036/team3-library'
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        KUBECONFIG_PATH = "${WORKSPACE}/kubeconfig"
        TERRAFORM_EXEC_PATH = "${terraform}"
        TERRAFORM_CONFIG_PATH = "${WORKSPACE}/terraform"
        AWS_CLI_PATH = "${aws}"
        KUBECTL_PATH = "${kubectl}"
        NAMESPACE = 'team3'
        TRIVY_PATH = "${trivy}"
        HELM_PATH = "${helm}"
        GRAFANA_ADMIN_PASSWORD = 'admin'
        PROMETHEUS_SCRAPE_INTERVAL = '30s'
        GITHUB_TOKEN = credentials('github-token')
        TRIVY_RESULTS_FILE = "results/trivy-results.txt"
        Python_path = "${python}"
        TERRASCAN_PATH = "${terrascan}"
        INFRACOST_PATH = "${infracost}"
        RESULTS_DIR = "${WORKSPACE}/results"
    }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('CI: Build and Test') {
            stages {
               stage('Create Results Directory') {
                    steps {
                        script {
                            sh 'mkdir -p results'
                        }
                    }
                }
                

                stage('Setup Python Environment') {
                    steps {
                        sh '''
                            set -eu
                            python3 -m venv .venv
                            .venv/bin/python -m pip install --upgrade pip
                            .venv/bin/python -m pip install -r requirements.txt
                            .venv/bin/python -m pip install flake8 black pytest pytest-cov coverage
                        '''
                    }
                }


                stage('Setup AWS CLI') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'aws-orange-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                           sh '''
                                aws configure set aws_access_key_id "$AWS_ACCESS_KEY_ID" --profile orange
                                aws configure set aws_secret_access_key "$AWS_SECRET_ACCESS_KEY" --profile orange
                                aws configure set region us-west-2 --profile orange
                        '''
                        }
                    }
                }

                stage('Build Docker Image') {
                    steps {
                        script {
                            echo "Building Docker image ${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                            sh "docker build -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} ."
                        }
                    }
                }

                stage('Code Quality Checks') {
                    stages {



                stage('Flake8') {
                    steps {
                        sh '''
                            set -e
                            .venv/bin/python -m black --line-length 100 app.py tests/
                            .venv/bin/python -m flake8 app.py tests/ \
                                --max-line-length=100 \
                                --output-file=results/flake8.log \
                                --tee
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'results/flake8.log',
                                allowEmptyArchive: true
                        }
                    }
                }




                stage('Black') {
                    steps {
                        sh '''
                            set -e
                            .venv/bin/python -m black --check --line-length 100 app.py tests/ \
                                2>&1 | tee results/black.log                                '''
                    }                        
                    post {
                        always {
                            archiveArtifacts artifacts: 'results/black.log',
                                allowEmptyArchive: true
                        }
                    }                                            
                }

                        
                            stage('Pytest') {
                                steps {
                                    sh '''
                                        .venv/bin/python -m pytest --junitxml=results/test-results.xml
                                    '''
                                }
                                post {
                                    always {
                                        archiveArtifacts artifacts: 'results/test-results.xml',
                                            allowEmptyArchive: true
                                        junit testResults: 'results/test-results.xml',
                                            allowEmptyResults: true
                                    }
                                }
                            }



                        stage('Unit Test Coverage') {
                            steps {
                                sh '''
                                    .venv/bin/python -m coverage run -m pytest
                                    .venv/bin/python -m coverage report -m > results/coverage.txt
                                    .venv/bin/python -m coverage xml -o results/coverage.xml
                                '''
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: 'results/coverage.*',
                                        allowEmptyArchive: true
                                }
                            }
                        }
                    }
                }




                    stage('Scan Docker Image with Trivy') {
                        steps {
                            script {
                                sh '''
                                    set -eu
                                    mkdir -p results
                                    trivy image \
                                        --format table \
                                        --output results/trivy-results.txt \
                                        "$DOCKER_IMAGE:$BUILD_NUMBER"
                                    cat results/trivy-results.txt
                                '''
                            }
                        }
                        post {
                            always {
                                archiveArtifacts(
                                    artifacts: 'results/trivy-results.txt',
                                    allowEmptyArchive: true
                                )
                            }
                        }
                    }


               


                stage('Security Scanning with Terrascan') {
                    steps {
                        dir("${env.TERRAFORM_CONFIG_PATH}") {
                            sh '''
                                mkdir -p "$WORKSPACE/results"
                                terrascan scan -d . -o json \
                                    > "$WORKSPACE/results/terrascan-report.json"
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts(
                                artifacts: 'results/terrascan-report.json',
                                allowEmptyArchive: true
                            )
                        }
                    }
                }



                stage('Infrastructure Cost Estimation with Infracost') {
                    steps {
                        dir("${env.TERRAFORM_CONFIG_PATH}") {
                            withCredentials([
                                string(
                                    credentialsId: 'infracost-api-key',
                                    variable: 'INFRACOST_API_KEY'
                                )
                            ]) {
                                sh '''
                                    mkdir -p "$WORKSPACE/results"
                                    infracost breakdown \
                                        --path . \
                                        --show-skipped \
                                        --out-file "$WORKSPACE/results/infracost-report.json"
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts(
                                artifacts: 'results/infracost-report.json',
                                allowEmptyArchive: true
                            )
                        }
                    }
                }

            }
        }

        stage('CD: Deploy') {
            stages {
                
                stage('Terraform Init and Apply') {
                    steps {
                        dir("${env.TERRAFORM_CONFIG_PATH}") {
                            sh '''
                                set -eu
                                export AWS_PROFILE=orange
                                terraform init
                                terraform validate
                                terraform plan -out=tfplan
                            '''
                        }
                    }
                }



                stage('Extract EKS Cluster Name') {
                    steps {
                        script {
                            dir("${env.TERRAFORM_CONFIG_PATH}") {
                                env.EKS_CLUSTER_NAME = sh(
                                    script: '"$TERRAFORM_EXEC_PATH" output -raw eks_cluster_name',
                                    returnStdout: true
                                ).trim()
                
                                echo "EKS Cluster Name is set to ${env.EKS_CLUSTER_NAME}"
                            }
                        }
                    }
                }



                stage('Configure Kubeconfig') {
                    steps {
                        script {
                            echo "Configuring kubeconfig for EKS Cluster: ${env.EKS_CLUSTER_NAME}"
                
                            sh '''
                                set -eu
                                aws eks update-kubeconfig \
                                    --region us-west-2 \
                                    --name "$EKS_CLUSTER_NAME" \
                                    --kubeconfig "$WORKSPACE/kubeconfig" \
                                    --profile orange
                            '''
                
                            env.KUBECONFIG = "${env.WORKSPACE}/kubeconfig"
                            echo "KUBECONFIG is set to ${env.KUBECONFIG}"
                        }
                    }
                }



                stage('Push Docker Image to Docker Hub') {
                    steps {
                        script {
                            echo "Pushing Docker image ${DOCKER_IMAGE}:${env.BUILD_NUMBER} to Docker Hub"
                
                            withCredentials([
                                usernamePassword(
                                    credentialsId: "${DOCKER_CREDENTIALS}",
                                    usernameVariable: 'DOCKER_USERNAME',
                                    passwordVariable: 'DOCKER_PASSWORD'
                                )
                            ]) {
                                sh '''
                                    set -eu
                                    echo "$DOCKER_PASSWORD" | docker login \
                                        --username "$DOCKER_USERNAME" \
                                        --password-stdin
                
                                    docker tag "$DOCKER_IMAGE:$BUILD_NUMBER" "$DOCKER_IMAGE:latest"
                                    docker push "$DOCKER_IMAGE:$BUILD_NUMBER"
                                    docker push "$DOCKER_IMAGE:latest"
                
                                    docker logout
                                '''
                            }
                        }
                    }
                }


                stage('Push Docker Image to GitHub Container Registry') {
                    steps {
                        script {
                            withCredentials([
                                string(
                                    credentialsId: 'github-token',
                                    variable: 'GHCR_TOKEN'
                                )
                            ]) {
                                sh '''
                                    set -eu
                
                                    echo "$GHCR_TOKEN" | docker login ghcr.io \
                                        --username Prachizz \
                                        --password-stdin
                
                                    docker tag "$DOCKER_IMAGE:$BUILD_NUMBER" \
                                        "$DOCKER_IMAGE_GHCR:$BUILD_NUMBER"
                
                                    docker push "$DOCKER_IMAGE_GHCR:$BUILD_NUMBER"
                
                                    docker logout ghcr.io
                                '''
                            }
                        }
                    }
                }



                stage('Update Kubernetes Manifests and Push Results') {
                    steps {
                        script {
                            // Checkout your forked repository
                            git(
                                url: 'https://github.com/Prachizz/Library-Inventory-System-DevOps.git',
                                branch: 'main',
                                changelog: false,
                                poll: false
                            )
                
                            // Update the Kubernetes deployment image
                            sh '''
                                set -eu
                
                                sed -i "s|^[[:space:]]*image:.*|          image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|" \
                                    k8s/deployment.yaml
                            '''
                
                            // Check whether Git detected any changes
                            def changes = sh(
                                script: 'git status --porcelain',
                                returnStdout: true
                            ).trim()
                
                            if (changes) {
                                echo 'Changes detected; committing updates.'
                
                                withCredentials([
                                    string(
                                        credentialsId: 'github-token',
                                        variable: 'GH_TOKEN'
                                    )
                                ]) {
                                    sh '''
                                        set -eu
                
                                        git config user.name "Jenkins CI"
                                        git config user.email "jenkins@orange.com"
                
                                        git add k8s/deployment.yaml
                
                                        if [ -d results ]; then
                                            git add results/
                                        fi
                
                                        if ! git diff --cached --quiet; then
                                            git commit -m "Update deployment image to ${BUILD_NUMBER}"
                
                                            git -c http.extraheader="AUTHORIZATION: basic $(printf 'x-access-token:%s' "$GH_TOKEN" | base64 | tr -d '\\n')" \
                                                push origin HEAD:main
                                        else
                                            echo "No staged changes to commit."
                                        fi
                                    '''
                                }
                            } else {
                                echo 'No changes to commit.'
                            }
                        }
                    }
                }



                stage('Deploy to Kubernetes') {
                    steps {
                        script {
                            echo "Deploying to Kubernetes namespace: ${NAMESPACE}"
                
                            sh '''
                                set -eu
                
                                export KUBECONFIG="$WORKSPACE/kubeconfig"
                
                                kubectl create namespace "$NAMESPACE" \
                                    --dry-run=client -o yaml | kubectl apply -f -
                
                                for file in \
                                    k8s/persistent-volume.yaml \
                                    k8s/persistent-volume-claim.yaml \
                                    k8s/deployment.yaml \
                                    k8s/service.yaml
                                do
                                    if [ ! -f "$WORKSPACE/$file" ]; then
                                        echo "ERROR: Manifest not found: $WORKSPACE/$file"
                                        exit 1
                                    fi
                                done
                
                                kubectl apply -f "$WORKSPACE/k8s/persistent-volume.yaml"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/persistent-volume-claim.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/deployment.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/service.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl get pods -n "$NAMESPACE"
                                kubectl get services -n "$NAMESPACE"
                            '''
                        }
                    }
                }



                stage('Deploy Grafana and Prometheus') {
                    steps {
                        script {
                            echo "Deploying Prometheus and Grafana to ${NAMESPACE}"
                
                            sh '''
                                set -eu
                
                                export KUBECONFIG="$WORKSPACE/kubeconfig"
                
                                files="
                                k8s/prometheus/prometheus-clusterrole.yaml
                                k8s/prometheus/prometheus-clusterrolebinding.yaml
                                k8s/monitoring/prometheus-pvc.yaml
                                k8s/prometheus/prometheus-config.yaml
                                k8s/prometheus/prometheus-deployment.yaml
                                k8s/prometheus/prometheus-service.yaml
                                k8s/grafana/grafana-deployment.yaml
                                k8s/grafana/grafana-service.yaml
                                "
                
                                for file in $files; do
                                    if [ ! -f "$WORKSPACE/$file" ]; then
                                        echo "ERROR: Manifest not found: $WORKSPACE/$file"
                                        exit 1
                                    fi
                                done
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/prometheus/prometheus-clusterrole.yaml"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/prometheus/prometheus-clusterrolebinding.yaml"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/monitoring/prometheus-pvc.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/prometheus/prometheus-config.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/prometheus/prometheus-deployment.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/prometheus/prometheus-service.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/grafana/grafana-deployment.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl apply \
                                    -f "$WORKSPACE/k8s/grafana/grafana-service.yaml" \
                                    -n "$NAMESPACE"
                
                                kubectl get pods -n "$NAMESPACE"
                            '''
                        }
                    }
                }


                // stage('Deploy with Helm') {
                //     steps {
                //         script {
                //             echo "Deploying Helm charts to Kubernetes namespace: ${NAMESPACE}"
                //             bat """
                //                 "${env.HELM_PATH}" repo add prometheus-community https://prometheus-community.github.io/helm-charts
                //                 "${env.HELM_PATH}" repo add grafana https://grafana.github.io/helm-charts
                //                 "${env.HELM_PATH}" repo update

                //                 # Check if Prometheus is already installed
                //                 "${env.HELM_PATH}" status prometheus --namespace ${NAMESPACE} >nul 2>&1 || (
                //                     "${env.HELM_PATH}" install prometheus prometheus-community/prometheus --namespace ${NAMESPACE} --set alertmanager.persistentVolume.enabled=false --set server.persistentVolume.enabled=false --set pushgateway.persistentVolume.enabled=false --set server.global.scrape_interval=${PROMETHEUS_SCRAPE_INTERVAL}
                //                 )

                //                 # Check if Grafana is already installed
                //                 "${env.HELM_PATH}" status grafana --namespace ${NAMESPACE} >nul 2>&1 || (
                //                     "${env.HELM_PATH}" install grafana grafana/grafana --namespace ${NAMESPACE} --set admin.password=${GRAFANA_ADMIN_PASSWORD} --set service.type=LoadBalancer
                //                 )
                //             """
                //         }
                //     }
                // }

                // stage('Apply Kubernetes Manifests') {
                //     steps {
                //         script {
                //             echo "Applying Kubernetes manifests in namespace: ${NAMESPACE}"
                //             bat """
                //                 "${env.KUBECTL_PATH}" apply -f ${env.WORKSPACE}\\k8s\\pv-prometheus-alertmanager.yaml -n ${NAMESPACE}
                //                 "${env.KUBECTL_PATH}" apply -f ${env.WORKSPACE}\\k8s\\prometheus-server-service.yaml -n ${NAMESPACE} --validate=false
                //                 "${env.KUBECTL_PATH}" apply -f ${env.WORKSPACE}\\k8s\\alert-rules.yml -n ${NAMESPACE} --validate=false

                //             """
                //         }
                //     }
                // }


            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'CI/CD pipeline failed.'
        }
        always {
            echo 'Cleaning up...'
        }
    }
}
