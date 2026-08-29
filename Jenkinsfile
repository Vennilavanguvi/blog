/*
  Production CI/CD pipeline: Blog App -> EKS
  Stages: checkout -> unit tests -> SonarQube -> Quality Gate -> build ->
          Nexus (artifact/npm) -> Docker build -> Trivy scan -> push to ECR ->
          deploy to EKS -> smoke test -> notify.

  Required Jenkins plugins: Pipeline, Docker Pipeline, SonarQube Scanner,
  Kubernetes CLI, AWS Steps, Slack Notification (optional).

  Required Jenkins credentials:
    - 'aws-creds'        (AWS access key/secret or use an IAM role on the agent)
    - 'sonarqube-token'  (SonarQube auth token)
    - 'nexus-creds'      (Nexus repo username/password)
    - 'ecr-registry'     (string param: <account>.dkr.ecr.<region>.amazonaws.com)
    - 'kubeconfig-eks'   (kubeconfig file credential, or use `aws eks update-kubeconfig`)
*/

pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    AWS_REGION       = 'ap-south-1'
    EKS_CLUSTER      = 'blogapp-eks-cluster'
    ECR_REGISTRY     = credentials('ecr-registry')
    NEXUS_URL        = 'http://nexus.internal:8081/repository/blogapp-npm'
    SONAR_PROJECT    = 'blogapp'
    IMAGE_TAG        = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'local'}"
    TRIVY_SEVERITY   = 'CRITICAL,HIGH'
    K8S_NAMESPACE    = 'blogapp'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install Dependencies') {
      parallel {
        stage('Backend deps') {
          steps { dir('backend') { sh 'npm ci' } }
        }
        stage('Frontend deps') {
          steps { dir('frontend') { sh 'npm ci' } }
        }
      }
    }

    stage('Unit Tests') {
      steps {
        dir('backend') { sh 'npm test -- --ci' }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube-server') {
          sh """
            sonar-scanner \
              -Dsonar.projectKey=${SONAR_PROJECT} \
              -Dsonar.sources=backend/src,frontend/src \
              -Dsonar.exclusions=**/node_modules/**,**/dist/** \
              -Dsonar.javascript.lcov.reportPaths=backend/coverage/lcov.info
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Publish Artifacts to Nexus') {
      steps {
        dir('backend') {
          sh 'npm pack'
        }
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            curl -u "$NEXUS_USER:$NEXUS_PASS" \
              --upload-file backend/*.tgz \
              "$NEXUS_URL/blogapp-backend/${IMAGE_TAG}/"
          '''
        }
      }
    }

    stage('Build Docker Images') {
      parallel {
        stage('Backend image') {
          steps {
            sh "docker build -t ${ECR_REGISTRY}/blogapp-backend:${IMAGE_TAG} ./backend"
          }
        }
        stage('Frontend image') {
          steps {
            sh "docker build -t ${ECR_REGISTRY}/blogapp-frontend:${IMAGE_TAG} ./frontend"
          }
        }
      }
    }

    stage('Trivy Vulnerability Scan') {
      parallel {
        stage('Scan backend image') {
          steps {
            sh """
              trivy image --exit-code 1 --severity ${TRIVY_SEVERITY} \
                --format table --ignore-unfixed \
                ${ECR_REGISTRY}/blogapp-backend:${IMAGE_TAG}
            """
          }
        }
        stage('Scan frontend image') {
          steps {
            sh """
              trivy image --exit-code 1 --severity ${TRIVY_SEVERITY} \
                --format table --ignore-unfixed \
                ${ECR_REGISTRY}/blogapp-frontend:${IMAGE_TAG}
            """
          }
        }
        stage('Filesystem scan (deps)') {
          steps {
            sh "trivy fs --exit-code 0 --severity ${TRIVY_SEVERITY} ."
          }
        }
      }
    }

    stage('Push Images to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh """
            aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${ECR_REGISTRY}
            docker push ${ECR_REGISTRY}/blogapp-backend:${IMAGE_TAG}
            docker push ${ECR_REGISTRY}/blogapp-frontend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh """
            aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

            kubectl create configmap mysql-init-script \
              --from-file=init.sql=database/init.sql \
              -n ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

            sed -i "s#<ECR_REGISTRY>/blogapp-backend:IMAGE_TAG#${ECR_REGISTRY}/blogapp-backend:${IMAGE_TAG}#" k8s/backend/deployment.yaml
            sed -i "s#<ECR_REGISTRY>/blogapp-frontend:IMAGE_TAG#${ECR_REGISTRY}/blogapp-frontend:${IMAGE_TAG}#" k8s/frontend/deployment.yaml

            kubectl apply -k k8s/

            kubectl rollout status deployment/backend -n ${K8S_NAMESPACE} --timeout=180s
            kubectl rollout status deployment/frontend -n ${K8S_NAMESPACE} --timeout=180s
          """
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh """
          ENDPOINT=\$(kubectl get ingress blogapp-ingress -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -sf "http://\${ENDPOINT}/api/posts?limit=1" || (echo "Smoke test failed" && exit 1)
        """
      }
    }
  }

  post {
    success {
      echo "Deployed blogapp build ${IMAGE_TAG} to EKS namespace ${K8S_NAMESPACE}."
    }
    failure {
      echo "Pipeline failed at stage: ${env.STAGE_NAME}. Rolling back is manual: kubectl rollout undo deployment/<name> -n ${K8S_NAMESPACE}"
    }
    always {
      sh 'docker image prune -f || true'
    }
  }
}
