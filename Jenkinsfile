pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-21
                    command: [sleep]
                    args: [infinity]
                    volumeMounts:
                    - name: m2-cache
                      mountPath: /root/.m2
                  - name: kaniko
                    image: gcr.io/kaniko-project/executor:debug
                    command: [sleep]
                    args: [infinity]
                  - name: kubectl
                    image: bitnami/kubectl:1.32
                    command: [sleep]
                    args: [infinity]
                  volumes:
                  - name: m2-cache
                    emptyDir: {}
            '''
            defaultContainer 'maven'
        }
    }

    environment {
        REGISTRY    = "registry.jenkins.svc.cluster.local:5000"
        IMAGE_NAME  = "demo-app"
        IMAGE_TAG   = "${BUILD_NUMBER}"
        NAMESPACE   = "jenkins"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                container('maven') {
                    sh 'mvn clean package -B'
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --context=`pwd` \
                          --dockerfile=`pwd`/Dockerfile \
                          --destination=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                          --insecure \
                          --skip-tls-verify
                    """
                }
            }
        }

        stage('Deploy to Cluster') {
            steps {
                container('kubectl') {
                    sh """
                        sed 's|IMAGE_TAG|${IMAGE_TAG}|g' k8s/deployment.yaml > /tmp/deployment-rendered.yaml
                        kubectl apply -f /tmp/deployment-rendered.yaml -n ${NAMESPACE}
                        kubectl rollout status deployment/demo-app -n ${NAMESPACE} --timeout=120s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${IMAGE_NAME}:${IMAGE_TAG} successfully to namespace ${NAMESPACE}"
        }
        failure {
            echo "Pipeline failed — check stage logs above"
        }
    }
}
