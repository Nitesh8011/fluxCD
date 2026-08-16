pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Helm lint') {
            steps {
                sh 'helm lint flux/helm/nginx-app'
            }
        }

        stage('Helm template') {
            steps {
                sh '''
                    helm template flux/helm/nginx-app -f flux/helm/nginx-app/values-staging.yaml > /dev/null
                    helm template flux/helm/nginx-app -f flux/helm/nginx-app/values-production.yaml > /dev/null
                '''
            }
        }

        stage('Kustomize build') {
            steps {
                sh '''
                    kubectl kustomize flux/kustomize/staging > /dev/null
                    kubectl kustomize flux/kustomize/production > /dev/null
                    kubectl kustomize clusters/staging > /dev/null
                    kubectl kustomize clusters/production > /dev/null
                '''
            }
        }
    }
}
