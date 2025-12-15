pipeline {
  agent any
  environment {
    AWS_DEFAULT_REGION = 'ap-south-1'
    ACCOUNT_ID = '${ACCOUNT_ID}'
    ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/nginx-app"
    GITOPS_REPO = "https://github.com/Vaishnavisousuddi/gitops-nginx.git"
    APP_PATH = "nginx-site"
  }
  options { timestamps() }

  stages {
    stage('Checkout Source Code') {
      steps {
        echo "📦 Checking out application source code..."
        git branch: 'main', url: 'https://github.com/Vaishnavisousuddi/argocd-jenkins-ci.git'
      }
    }

    stage('Build & Push Image to ECR') {
      steps {
        echo "🔨 Building and pushing Docker image to ECR..."
        withAWS(credentials: 'aws-creds', region: "${AWS_DEFAULT_REGION}") {
          script {
            IMAGE_TAG = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()
            sh """
              aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REPO}
              docker build -t ${ECR_REPO}:${IMAGE_TAG} .
              docker push ${ECR_REPO}:${IMAGE_TAG}
            """
            // ✅ Persist IMAGE_TAG for next stage
            writeFile file: 'image_tag.txt', text: IMAGE_TAG
          }
        }
      }
    }

    stage('Update Helm Values in GitOps Repo') {
      steps {
        echo "🧩 Updating Helm chart with new image tag..."
        withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          script {
            IMAGE_TAG = readFile('image_tag.txt').trim()
            sh """
              rm -rf gitops-nginx
              git clone https://${GH_USER}:${GH_TOKEN}@github.com/Vaishnavisousuddi/gitops-nginx.git
              cd gitops-nginx/${APP_PATH}
              sed -i "s|tag:.*|tag: \"${IMAGE_TAG}\"|" values.yaml
              git config user.name "jenkins"
              git config user.email "jenkins@local"
              git commit -am "deploy: image tag ${IMAGE_TAG}"
              git push
            """
          }
        }
      }
    }

    stage('Trigger ArgoCD Sync (optional)') {
      steps {
        echo "🚀 ArgoCD auto-sync will detect the new tag and deploy automatically."
      }
    }
  }

  post {
    failure {
      echo "❌ Pipeline failed. Check logs for details."
    }
  }
}
