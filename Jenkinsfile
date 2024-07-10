pipeline {
    agent any
    
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'asia-east1-docker.pkg.dev/charged-gravity-425405-j6/docker-image/java-application')
        string(name: 'NAMESPACE', defaultValue: 'shram')
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
                    def result = sh(returnStdout: true, script: "gcloud artifacts docker images list ${params.IMAGE_NAME} --include-tags")
                    if (result.trim() != "") {
                        echo "Image ${params.IMAGE_NAME} exists in Artifact Registry!"
                    } else {
                        error "Image ${params.IMAGE_NAME} does not exist in Artifact Registry. Aborting deployment."
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
                sh 'kubectl apply -f Deployment.yml -f ingress.yml -n ${NAMESPACE}'
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
