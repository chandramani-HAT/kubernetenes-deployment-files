pipeline {
  agent { label 'master' }

  environment {
    AWS_REGION = 'us-east-1'
    EKS_CLUSTER_NAME = 'your-eks-cluster'
    EKS_NAMESPACE = 'document-expert'
    ECR_REGISTRY = '028892270743.dkr.ecr.us-east-1.amazonaws.com'
    ECR_SECRET_NAME = 'ecr-creds'
    HOME = '/home/ubuntu'
    KUBECONFIG = "${HOME}/.kube/config"
    CLUSTER_NAME = 'kubernetes'
  }

  stages {
    stage('Checkout') {
      steps {
        git(
        branch: 'main',
        credentialsId: 'github-repo',
        url: 'https://github.com/chandramani-HAT/kubernetenes-deployment-files.git'
        )
      }
    }

    stage('Test kubectl') {
      steps {
        sh 'kubectl get nodes'
      }
    }

    stage('Create namespaces') {
      steps {
        sh 'kubectl apply -f namespace.yaml'
      }
    }    



    stage('Create ECR imagePullSecret') {
      steps {
        script {
          // Delete old secret if it exists (ignore error)
          sh "kubectl delete secret ${ECR_SECRET_NAME} -n ${EKS_NAMESPACE} || true"
          // Create new secret with current ECR token
          sh '''
            PASSWORD=$(aws ecr get-login-password --region $AWS_REGION)
            kubectl create secret docker-registry $ECR_SECRET_NAME \
              --docker-server=$ECR_REGISTRY \
              --docker-username=AWS \
              --docker-password="$PASSWORD" \
              --namespace=$EKS_NAMESPACE
          '''
        }
      }
    }

    stage('Install Helm') {
      steps {
        sh '''
          curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
          chmod 700 get_helm.sh
          ./get_helm.sh
          helm version
        '''
      }
    }

    stage('Install External Secrets Operator') {
      steps {
        sh '''
          helm repo add external-secrets https://charts.external-secrets.io
          helm repo update
          if helm list -n external-secrets --filter '^external-secrets$' | grep external-secrets; then
            echo "external-secrets release already exists, skipping installation."
          else
            helm install external-secrets external-secrets/external-secrets \
              -n external-secrets \
              --create-namespace \
              --set installCRDs=true
          fi
          kubectl get crds | grep external-secrets
          # Wait for CRDs to be registered
          sleep 15
          # Wait for operator pods to be ready
          kubectl rollout status deployment/external-secrets -n external-secrets --timeout=120s || true
          kubectl get pods -n external-secrets
          kubectl get crds | grep external-secrets
        '''
      }
    }

    stage('Apply External Secrets') {
      steps {
          sh 'kubectl apply -f external-secrets/ -n $EKS_NAMESPACE'
        }
    }

    stage('Apply Postgres Manifest') {
      steps {
          sh '''
            kubectl apply -f postgres/ -n $EKS_NAMESPACE
            kubectl rollout status deployment/postgres -n $EKS_NAMESPACE || true
          '''
        }
    }

    stage('Apply Redis Manifest') {
      steps {
          sh '''
            kubectl apply -f redis/ -n $EKS_NAMESPACE
            kubectl rollout status deployment/redis -n $EKS_NAMESPACE || true
          '''
        }
    }

    stage('Apply Celery Manifest') {
      steps {
          sh '''
            kubectl apply -f backend/ -n $EKS_NAMESPACE
            kubectl rollout status deployment/celery -n $EKS_NAMESPACE || true
          '''
      }
    }

    stage('Apply Backend Manifest') {
      steps {
          sh '''
            kubectl apply -f backend/ -n $EKS_NAMESPACE
            kubectl rollout status deployment/backend -n $EKS_NAMESPACE || true
          '''
      }
    }

    stage('Apply Frontend Manifest') {
      steps {
          sh '''
            kubectl apply -f frontend/ -n $EKS_NAMESPACE
            kubectl rollout status deployment/frontend -n $EKS_NAMESPACE || true
          '''
        }
    }

    stage('Install cert-manager') {
      steps {
        sh '''
          kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.3/cert-manager.yaml
          # Wait for cert-manager to be ready
          kubectl rollout status deployment/cert-manager -n cert-manager --timeout=180s || true
        '''
      }
    }
    stage('Add Helm repo for AWS Load Balancer Controller') {
      steps {
        sh '''
          helm repo add eks https://aws.github.io/eks-charts
          helm repo update
        '''
      }
    }
    stage('Install/Upgrade AWS Load Balancer Controller') {
      steps {
        sh '''
          helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
            -n kube-system \
            --set clusterName=$CLUSTER_NAME \
            --set region=$AWS_REGION
        '''
      }
    }
    stage('Verify Controller Deployment') {
      steps {
        sh 'kubectl get deployment -n kube-system aws-load-balancer-controller'
        sh 'kubectl get pods -n kube-system | grep aws-load-balancer-controller'
      }
    }

    stage('Apply Ingress class  Manifest') {
      steps {
          sh 'kubectl apply -f ingress-class.yaml -n $EKS_NAMESPACE'
        }
      }

    stage('Apply Ingress Manifest') {
      steps {
          sh 'kubectl apply -f ingress.yaml -n $EKS_NAMESPACE'
        }
      }
  }
}
