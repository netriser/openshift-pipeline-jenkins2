pipeline {
    agent any
    parameters {
        string(name: 'PROJECT_NAME', defaultValue: 'mon-projet', description: 'Nom du projet OpenShift')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0.0', description: 'Version de tag de l\'image (ne doit pas être "latest")')
    }
    environment {
        OPENSHIFT_API_URL = 'https://api.crc.testing:6443'
        OPENSHIFT_TOKEN = credentials('openshift-token') // ID du Credential Jenkins
        OPENSHIFT_PROJECT = "${params.PROJECT_NAME}"
        IMAGE_TAG = "${params.IMAGE_TAG}"
    }
    stages {
        stage('Setup Environment') {
            steps {
                script {
                    withEnv(["PATH+OC=${tool 'oc3.11'}", "KUBECONFIG=${env.WORKSPACE}/kubeconfig"]) {
                        sh """
                        echo 'Initializing KUBECONFIG and OpenShift CLI...'
                        oc login ${OPENSHIFT_API_URL} --token=${OPENSHIFT_TOKEN} --insecure-skip-tls-verify
                        
                        echo 'Checking if project ${OPENSHIFT_PROJECT} exists...'
                        if ! oc get project ${OPENSHIFT_PROJECT} > /dev/null 2>&1; then
                            echo 'Project does not exist. Creating project ${OPENSHIFT_PROJECT}...'
                            oc new-project ${OPENSHIFT_PROJECT}
                        else
                            echo 'Project ${OPENSHIFT_PROJECT} already exists.'
                        fi
                        oc project ${OPENSHIFT_PROJECT}
                        """
                    }
                }
            }
        }
        stage('Create BuildConfig') {
            steps {
                script {
                    withEnv(["PATH+OC=${tool 'oc3.11'}", "KUBECONFIG=${env.WORKSPACE}/kubeconfig"]) {
                        sh """
                        echo 'Checking if BuildConfig exists...'
                        if ! oc get bc html-nginx-build > /dev/null 2>&1; then
                            echo 'BuildConfig does not exist. Creating BuildConfig...'
                            oc new-build --name=html-nginx-build --binary --strategy=source --image-stream="openshift/nginx:latest"
                        else
                            echo 'BuildConfig already exists.'
                        fi
                        """
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    if (params.IMAGE_TAG == 'latest') {
                        error "Image tag cannot be 'latest'. Please provide a specific version for IMAGE_TAG."
                    }
                    withEnv(["PATH+OC=${tool 'oc3.11'}", "KUBECONFIG=${env.WORKSPACE}/kubeconfig"]) {
                        sh """
                        echo 'Starting OpenShift build in project ${OPENSHIFT_PROJECT} with tag ${IMAGE_TAG}...'
                        oc start-build html-nginx-build --from-dir=. --follow
                        """
                    }
                }
            }
        }
        stage('Deploy to OpenShift') {
            steps {
                script {
                    withEnv(["PATH+OC=${tool 'oc3.11'}", "KUBECONFIG=${env.WORKSPACE}/kubeconfig"]) {
                        sh """
                        echo 'Deploying application in project ${OPENSHIFT_PROJECT}...'
                        oc project ${OPENSHIFT_PROJECT}
                        if oc get dc html-nginx; then
                            echo 'DeploymentConfig exists, rolling out...'
                            oc set image dc/html-nginx html-nginx=html-nginx:${IMAGE_TAG}
                            oc rollout status dc/html-nginx
                        else
                            echo 'Creating new application with tag ${IMAGE_TAG}...'
                            oc new-app html-nginx-build:${IMAGE_TAG}
                        fi
                        """
                    }
                }
            }
        }
        stage('Expose Route') {
            steps {
                script {
                    withEnv(["PATH+OC=${tool 'oc3.11'}", "KUBECONFIG=${env.WORKSPACE}/kubeconfig"]) {
                        sh """
                        echo 'Exposing service route in project ${OPENSHIFT_PROJECT}...'
                        oc project ${OPENSHIFT_PROJECT}
                        if oc get route html-nginx-build > /dev/null 2>&1; then
                            echo 'Route already exists.'
                        else
                            if ! oc get svc html-nginx-build > /dev/null 2>&1; then
                                echo 'Service does not exist, creating service...'
                                oc expose dc/html-nginx-build
                            fi
                            oc expose svc/html-nginx-build
                            echo 'Route exposed successfully.'
                        fi
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            sh """
            echo 'Cleaning up KUBECONFIG...'
            rm -f ${env.WORKSPACE}/kubeconfig
            """
        }
    }
}
