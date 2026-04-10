pipeline {
    agent any

    environment {
        PROJECT_ID   = "cloud-computing-488905"
        REGION       = "asia-south1"
        ZONE         = "asia-south1-c"
        CLUSTER_NAME = "banking-gke"
        REGISTRY     = "gcr.io/${PROJECT_ID}"
    }

    stages {

        /******************************
         * 1. CHECKOUT SOURCE CODE
         *****************************/
        stage('Checkout Banking Source Code') {
            steps {
                dir('banking') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/main"]],
                        userRemoteConfigs: [[
                            url: "https://github.com/g25ai1020-lab/banking.git",
                            credentialsId: "GITHUB_PAT"
                        ]]
                    ])
                }
            }
        }

        /******************************
         * 2. CHECKOUT K8 YAML REPO
         *****************************/
        stage('Checkout Kubernetes Manifests') {
            steps {
                dir('k8s') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/master"]],
                        userRemoteConfigs: [[
                            url: "https://github.com/g25ai1020-lab/k8.git",
                            credentialsId: "GITHUB_PAT"
                        ]]
                    ])
                }
            }
        }

        /******************************
         * 3. AUTHENTICATE TO GOOGLE CLOUD
         ***************************
        stage('Authenticate to Google Cloud') {
            steps {
                withCredentials([file(credentialsId: 'GCP_SA_KEY_JSON', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project ${PROJECT_ID}
                        gcloud config set compute/zone ${ZONE}
                    '''
                }
            }
        }*/
        /******************************
         * 4. BUILD ALL MICROSERVICES
         ****************************/
        stage('Build All Services (Maven)') {
            steps {
                sh '''
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/Account-Service/pom.xml clean package -DskipTests
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/Fund-Transfer/pom.xml clean package -DskipTests
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/Sequence-Generator/pom.xml clean package -DskipTests
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/Service-Registry/pom.xml clean package -DskipTests
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/Transaction-Service/pom.xml clean package -DskipTests
                  /opt/jenkins/apache-maven-3.9.14/bin/mvn -f banking/User-Service/pom.xml clean package -DskipTests
                '''
            }
        }

        /******************************
         * 5. BUILD DOCKER IMAGES
         ****************************/
        stage('Build Docker Images') {
            steps {
                sh '''
                  docker build -t $REGISTRY/account-service:latest banking/Account-Service
                  docker build -t $REGISTRY/fund-transfer:latest banking/Fund-Transfer
                  docker build -t $REGISTRY/sequence-generator:latest banking/Sequence-Generator
                  docker build -t $REGISTRY/service-registry:latest banking/Service-Registry
                  docker build -t $REGISTRY/transaction-service:latest banking/Transaction-Service
                  docker build -t $REGISTRY/user-service:latest banking/User-Service
                '''
            }
        }

        /******************************
         * 6. PUSH IMAGES TO GCR
         ****************************/
        stage('Push Images to GCR') {
            steps {
                sh '''
                  gcloud auth configure-docker -q

                  docker push $REGISTRY/account-service:latest
                  docker push $REGISTRY/fund-transfer:latest
                  docker push $REGISTRY/sequence-generator:latest
                  docker push $REGISTRY/service-registry:latest
                  docker push $REGISTRY/transaction-service:latest
                  docker push $REGISTRY/user-service:latest
                '''
            }
        }

        /******************************
         * 7. CONNECT TO GKE
         *****************************/
        stage('Connect to GKE') {
            steps {
                sh '''
                  gcloud container clusters get-credentials ${CLUSTER_NAME} \
                    --zone ${ZONE} \
                    --project ${PROJECT_ID}
                '''
            }
        }

        /******************************
         * 8. DEPLOY ALL K8s SERVICES
         *****************************/
        stage('Deploy All Services to Kubernetes') {
            steps {
                sh '''
                  kubectl apply -f k8s/namespace.yaml

                  kubectl apply -f k8s/mysql/

                  kubectl apply -f k8s/service-registry.yaml
                  kubectl apply -f k8s/account-service.yaml
                  kubectl apply -f k8s/fund-transfer.yaml
                  kubectl apply -f k8s/sequence-generator.yaml
                  kubectl apply -f k8s/transaction-service.yaml
                  kubectl apply -f k8s/user-service.yaml
                  kubectl apply -f k8s/api-gateway.yaml
                  kubectl apply -f k8s/ingress.yaml
                '''
            }
        }

        /******************************
         * 9. VERIFY DEPLOYMENT
         *****************************/
        stage('Verify Deployment') {
            steps {
                sh '''
                  kubectl get pods -n banking-app
                  kubectl get svc  -n banking-app
                  kubectl get ingress -n banking-app
                '''
            }
        }
    }

    post {
        success {
            echo "✅ ALL microservices deployed successfully to GKE"
        }
        failure {
            echo "❌ Deployment failed – check Jenkins logs"
        }
    }
}
