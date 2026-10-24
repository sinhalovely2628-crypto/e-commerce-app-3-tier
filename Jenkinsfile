pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        WEB_IMAGE = 'ecom-web'
        DB_IMAGE = 'ecom-db'
        K8S_NAMESPACE = 'ecom'
        WORKER_PRIVATE_IP = '10.0.1.10'
    }

    stages {
        stage('1. Checkout Source Code') {
            steps {
                checkout scm

                sh '''
                    set -eu
                    test -f Dockerfile.web
                    test -f Dockerfile.db
                    test -f k8s_yaml/Namespace.yaml
                    test -f k8s_yaml/config.yaml
                    test -f k8s_yaml/secret.yaml
                    test -f k8s_yaml/ecom-db-statefulset.yaml
                    test -f k8s_yaml/ecom-web-deployment.yaml
                    test -f k8s_yaml/ecom-web-service.yaml
                    test -f k8s_yaml/ecommerce-ingress.yaml

                    echo "Repository and required files verified."
                '''
            }
        }

        stage('2. Verify Tools') {
            steps {
                sh '''
                    set -eu
                    docker --version
                    docker info >/dev/null
                    kubectl version --client
                    trivy --version
                    python3 --version
                '''
            }
        }

        stage('3. Build Docker Images') {
            steps {
                script {
                    env.IMAGE_TAG = "b${env.BUILD_NUMBER}"
                }

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -eu

                        docker build \
                          -f Dockerfile.web \
                          -t "$DOCKER_USER/ecom-web:$IMAGE_TAG" .

                        docker build \
                          -f Dockerfile.db \
                          -t "$DOCKER_USER/ecom-db:$IMAGE_TAG" .

                        echo "Both images built."
                    '''
                }
            }
        }

        // stage('4. Trivy Vulnerability Scan') {
        //     steps {
        //         withCredentials([
        //             usernamePassword(
        //                 credentialsId: 'dockerhub-creds',
        //                 usernameVariable: 'DOCKER_USER',
        //                 passwordVariable: 'DOCKER_PASS'
        //             )
        //         ]) {
        //             sh '''
        //                 set -eu

        //                 trivy image \
        //                   --scanners vuln \
        //                   --severity HIGH,CRITICAL \
        //                   --ignore-unfixed \
        //                   --exit-code 1 \
        //                   "$DOCKER_USER/ecom-web:$IMAGE_TAG"

        //                 trivy image \
        //                   --scanners vuln \
        //                   --severity HIGH,CRITICAL \
        //                   --ignore-unfixed \
        //                   --exit-code 1 \
        //                   "$DOCKER_USER/ecom-db:$IMAGE_TAG"
        //             '''
        //         }
        //     }
        // }

        stage('5. Push Images to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        echo "$DOCKER_PASS" |
                          docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin

                        docker push "$DOCKER_USER/ecom-web:$IMAGE_TAG"
                        docker push "$DOCKER_USER/ecom-db:$IMAGE_TAG"

                        docker logout
                    '''
                }
            }
        }

        stage('6. Update Kubernetes YAML Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -eu

                        export WEB_FULL_IMAGE="$DOCKER_USER/ecom-web:$IMAGE_TAG"
                        export DB_FULL_IMAGE="$DOCKER_USER/ecom-db:$IMAGE_TAG"

                        python3 <<'PY'
import os
import re

files = {
    "k8s_yaml/ecom-web-deployment.yaml": (
        "ecom-web", os.environ["WEB_FULL_IMAGE"]
    ),
    "k8s_yaml/ecom-db-statefulset.yaml": (
        "ecom-db", os.environ["DB_FULL_IMAGE"]
    ),
}

for path, (image_name, new_image) in files.items():
    with open(path) as f:
        content = f.read()

    pattern = (
        r"(?m)^(\\s*image:\\s*)"
        r"(?:[^\\s]*/)?"
        + re.escape(image_name)
        + r":[^\\s]+\\s*$"
    )

    updated, count = re.subn(
        pattern,
        lambda m: m.group(1) + new_image,
        content
    )

    if count == 0:
        raise SystemExit(
            f"Could not find an image reference for {image_name} "
            f"in {path}. Check the manifest's container image name."
        )

    with open(path, "w") as f:
        f.write(updated)

    print(f"Updated image in {path}: {new_image}")
PY

                        grep -n 'image:' \
                          k8s_yaml/ecom-web-deployment.yaml \
                          k8s_yaml/ecom-db-statefulset.yaml
                    '''
                }
            }
        }

        stage('7. Connect to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl cluster-info
                        kubectl get nodes -o wide
                    '''
                }
            }
        }

        stage('8. Apply Namespace, Config and Secrets') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    ),
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl apply -f k8s_yaml/Namespace.yaml

                        kubectl apply \
                          -n "$K8S_NAMESPACE" \
                          -f k8s_yaml/config.yaml

                        kubectl apply \
                          -n "$K8S_NAMESPACE" \
                          -f k8s_yaml/secret.yaml

                        # Allows Kubernetes to pull private Docker Hub images.
                        kubectl create secret docker-registry dockerhub-creds \
                          -n "$K8S_NAMESPACE" \
                          --docker-server=https://index.docker.io/v1/ \
                          --docker-username="$DOCKER_USER" \
                          --docker-password="$DOCKER_PASS" \
                          --dry-run=client -o yaml |
                          kubectl apply -f -

                        kubectl patch serviceaccount default \
                          -n "$K8S_NAMESPACE" \
                          --type=merge \
                          -p '{"imagePullSecrets":[{"name":"dockerhub-creds"}]}'
                    '''
                }
            }
        }

        stage('9. Select server2 Worker Node') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        WORKER_NODE=$(
                          kubectl get nodes -o json |
                          python3 -c '
import json, sys
data = json.load(sys.stdin)
for node in data["items"]:
    status = node.get("status", {})
    labels = node.get("metadata", {}).get("labels", {})
    addresses = status.get("addresses", [])
    ips = [
        a["address"] for a in addresses
        if a["type"] == "InternalIP"
    ]
    ready = any(
        c["type"] == "Ready" and c["status"] == "True"
        for c in status.get("conditions", [])
    )
    control_plane = (
        "node-role.kubernetes.io/control-plane" in labels
        or "node-role.kubernetes.io/master" in labels
    )
    if "10.0.1.10" in ips and ready and not control_plane:
        print(node["metadata"]["name"])
        break
'
                        )

                        if [ -z "$WORKER_NODE" ]; then
                            echo "server2 at 10.0.1.10 was not found as a Ready worker."
                            kubectl get nodes -o wide
                            exit 1
                        fi

                        echo "Selected worker node: $WORKER_NODE"

                        kubectl label node "$WORKER_NODE" \
                          ecom-worker=true --overwrite
                    '''
                }
            }
        }

        stage('10. Deploy Database and Web Application') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        # Deploy the database first.
                        kubectl apply \
                          -f k8s_yaml/ecom-db-statefulset.yaml \
                          -n "$K8S_NAMESPACE"

                        kubectl patch statefulset ecom-db \
                          -n "$K8S_NAMESPACE" \
                          --type=merge \
                          -p '{"spec":{"template":{"spec":{"nodeSelector":{"ecom-worker":"true"}}}}}'

                        kubectl rollout status statefulset/ecom-db \
                          -n "$K8S_NAMESPACE" \
                          --timeout=300s

                        # Deploy the web application and service.
                        kubectl apply \
                          -f k8s_yaml/ecom-web-deployment.yaml \
                          -n "$K8S_NAMESPACE"

                        kubectl patch deployment ecom-web \
                          -n "$K8S_NAMESPACE" \
                          --type=merge \
                          -p '{"spec":{"template":{"spec":{"nodeSelector":{"ecom-worker":"true"}}}}}'

                        kubectl apply \
                          -f k8s_yaml/ecom-web-service.yaml \
                          -n "$K8S_NAMESPACE"

                        kubectl rollout status deployment/ecom-web \
                          -n "$K8S_NAMESPACE" \
                          --timeout=300s
                    '''
                }
            }
        }

        stage('11. Deploy Ingress') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl apply -f k8s_yaml/ecommerce-ingress.yaml -n "$K8S_NAMESPACE"
                        kubectl get ingress -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }

        stage('12. Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "===== Pods and worker placement ====="
                        kubectl get pods \
                          -n "$K8S_NAMESPACE" -o wide

                        echo "===== Services ====="
                        kubectl get services -n "$K8S_NAMESPACE"

                        echo "===== Persistent Volumes ====="
                        kubectl get pvc -n "$K8S_NAMESPACE"

                        echo "===== Web deployment ====="
                        kubectl get deployment ecom-web \
                          -n "$K8S_NAMESPACE"

                        echo "===== Database StatefulSet ====="
                        kubectl get statefulset ecom-db \
                          -n "$K8S_NAMESPACE"

                        echo "===== Worker node ====="
                        kubectl get nodes -l ecom-worker=true -o wide

                        echo "===== Deployed web image ====="
                        kubectl get deployment ecom-web \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.spec.template.spec.containers[*].image}'
                        echo

                        echo "===== Deployed DB image ====="
                        kubectl get statefulset ecom-db \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.spec.template.spec.containers[*].image}'
                        echo
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS: Build ${env.BUILD_NUMBER} deployed to Kubernetes."
            echo "Image tag: ${env.IMAGE_TAG}"
        }

        failure {
            echo 'DEPLOYMENT FAILED. Check the first failed stage in Console Output.'
        }

        always {
            sh '''
                docker logout >/dev/null 2>&1 || true
                docker image prune -f >/dev/null 2>&1 || true
            '''
        }
    }
}
