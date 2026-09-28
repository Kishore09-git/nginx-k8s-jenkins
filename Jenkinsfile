pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'kishoredocker0912'
        IMAGE_NAME      = 'nginx-demo'
        FULL_IMAGE      = "kishoredocker0912/nginx-demo:${BUILD_NUMBER}"
        LATEST_IMAGE    = "kishoredocker0912/nginx-demo:latest"
        KUBECONFIG      = '/var/lib/jenkins/.kube/config'
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Build Image') {
            steps {
                dir('Build') {
                    sh 'docker build -t $FULL_IMAGE -t $LATEST_IMAGE .'
                    sh 'docker images | grep $IMAGE_NAME'
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                    sh 'docker push $FULL_IMAGE'
                    sh 'docker push $LATEST_IMAGE'
                    sh 'docker logout'
                }
            }
        }

        stage('Load Image into kind') {
            steps {
                sh 'kind load docker-image $FULL_IMAGE --name ci'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('Deploy') {
                    sh 'sed "s|IMAGE_TAG|${BUILD_NUMBER}|g" deployment.yaml > deployment.rendered.yaml'
                    sh 'grep "image:" deployment.rendered.yaml'
                    sh 'kubectl apply -f deployment.rendered.yaml'
                }
            }
        }

        stage('Verify Rollout') {
            steps {
                sh 'kubectl rollout status deployment/nginx-demo --timeout=180s'
                sh 'kubectl get deploy,po,svc -l app=nginx-demo -o wide'
                sh 'kubectl get endpoints nginx-demo'
                sh 'kubectl get deploy nginx-demo -o jsonpath="{.spec.template.spec.containers[0].image}"; echo'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
         	 for i in $(seq 1 30); do
                     ALB=$(kubectl get ingress nginx-demo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    		     [ -n "$ALB" ] && break
           	     sleep 10
                 done
                 echo "ALB: $ALB"
         	 curl -sf --max-time 15 --retry 10 --retry-delay 15 --retry-connrefused "http://$ALB:8080" | head -5
       	       '''
            }
        }
    }

    post {
        failure {
            echo 'Deploy failed - rolling back'
            sh 'kubectl rollout undo deployment/nginx-demo || true'
            sh 'kubectl rollout status deployment/nginx-demo --timeout=120s || true'
        }
        always {
            sh 'rm -f Deploy/deployment.rendered.yaml || true'
            sh 'docker image prune -f || true'
        }
    }
}
