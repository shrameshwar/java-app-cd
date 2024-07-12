pipeline {
    agent any
    
    parameters {
        string(name: 'Image_name')
        string(name: 'Tags')
        string(name: 'NAMESPACE')
    }
    
    environment {
        GCLOUD_PROJECT = 'charged-gravity-425405-j6'
        CLUSTER_NAME = 'cluster-1'
        CLUSTER_ZONE = 'us-central1-c'
        GCLOUD_CREDS = credentials('google_service_account_key_id') // Ensure 'google' is the correct credentials ID
    }
    
    stages {
        stage('gitpull') {
            steps {
                git url: 'https://github.com/shrameshwar/java-app-cd.git', branch: 'main'
            }
        }
        
        stage('GCP authentication') {
            steps {
                withCredentials([file(credentialsId: 'google_service_account_key_id', variable: 'GCLOUD_CREDS')]) {
                    sh '''
                    gcloud version
                    gcloud auth activate-service-account --key-file="$GCLOUD_CREDS"
                    gcloud config set project $GCLOUD_PROJECT
                    '''
                }
            }
        }
        
        stage('Artifact image existance') {
            steps {
                script {
                    def result = sh(returnStdout: true, script: "gcloud artifacts files list --project=charged-gravity-425405-j6 --repository=docker-image --location=asia-east1 --package=${params.Image_name} --tag=${params.Tags}")
                    if (result.trim() != "") {
                        echo "Image asia-east1-docker.pkg.dev/charged-gravity-425405-j6/docker-image/${params.Image_name}:${params.Tags} exists in Artifact Registry!"
                    } else {
                        error "Image asia-east1-docker.pkg.dev/charged-gravity-425405-j6/docker-image/${params.Image_name}:${params.Tags} does not exist in Artifact Registry. Aborting deployment."
                    }
                }
            }
        }
        
        stage('Connecting to the GKE Cluster') {
            steps {
                sh '''
                gcloud auth activate-service-account --key-file="$GCLOUD_CREDS"
                gcloud config set compute/zone $CLUSTER_ZONE
                gcloud container clusters get-credentials $CLUSTER_NAME
                '''
            }
        }
        
        stage('Namespace creation') {
            steps {
                script {
                    def exists = sh(script: "kubectl get namespace ${NAMESPACE} -o yaml", returnStatus: true) == 0
                    if (exists) {
                        echo "Namespace '${NAMESPACE}' already exists."
                    } else {
                        sh(script: "kubectl create namespace ${NAMESPACE}", returnStatus: true)
                        echo "Namespace '${NAMESPACE}' created."
                    }
                }
            }
        }
        stage('Deploy to GKE'){
            steps{
                sh '''
                sed -i 's|\${Image_name}|${params.Image_name}|g' Deployment.yml
                sed -i 's|\${Tags}|${params.Tags}|g' Deployment.yml
                kubectl apply -f Deployment.yml-n ${NAMESPACE}
                kubectl apply -f ingress.yml -n ${NAMESPACE}
                '''
            }
        }
    }
    post {
        success {
            echo 'Pipeline succeeded!!'
            mail to: 'shrameshwar1999@gmail.com',
                 subject: "Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                 body: "The pipeline ${env.BUILD_URL} has successfully completed."
        }
        failure {
            echo 'Pipeline failed!'
            mail to: 'shrameshwar1999@gmail.com',
                 subject: "Pipeline Failed: ${currentBuild.fullDisplayName}",
                 body: "The pipeline ${env.BUILD_URL} has failed. Check the logs for details."
        }
    }
}
