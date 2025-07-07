pipeline {
    agent any
    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        dockerImagePrefix = "us-west1-docker.pkg.dev/lab-agibiz/docker-repository"
        registry = "https://us-west1-docker.pkg.dev"
        registryCredentials = "gcp-registry"
    }
    stages {
        stage("Inicio ultima tarea") {
            steps {
                sh 'echo "comenzando mi pipeline"'
            }
        }
        stage("proceso de build y test") {
            agent {
                docker {
                    image 'node:22'
                    reuseNode true
                }
            }
            stages {
                stage("instalacion de dependencias") {
                    steps {
                        sh 'npm ci'
                    }
                }
                stage("ejecucion de pruebas") {
                    steps {
                        sh 'npm run test:cov'
                    }
                }
                stage("construccion de la aplicacion") {
                    steps {
                        sh 'npm run build'
                    }
                }
            }
        }
        stage("build y push de imagen docker") {
            steps {
                script {
                    docker.withRegistry("${registry}", registryCredentials) {
                        sh "docker build -t backend-nest-test-uac ."
                        sh "docker tag backend-nest-test-uac ${dockerImagePrefix}/backend-nest-test-uac"
                        sh "docker tag backend-nest-test-uac ${dockerImagePrefix}/backend-nest-test-uac:${BUILD_NUMBER}"
                        sh "docker push ${dockerImagePrefix}/backend-nest-test-uac"
                        sh "docker push ${dockerImagePrefix}/backend-nest-test-uac:${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage("actualizacion de kubernetes") {
            agent {
                docker {
                    image 'alpine/k8s:1.30.2'
                    reuseNode true
                }
            }
            steps {
                withKubeConfig([credentialsId: 'gcp-kubeconfig']) {
                    script {
                        // Validar y crear namespace si no existe
                        sh '''
                            if ! kubectl get namespace lab-uac >/dev/null 2>&1; then
                                echo "Namespace lab-uac no existe. Creando..."
                                kubectl create namespace lab-uac
                            else
                                echo "Namespace lab-uac ya existe."
                            fi
                        '''

                        // Validar si el deployment existe antes de hacer set image
                        sh """
                            if kubectl -n lab-uac get deployment backend-nest-test-uac >/dev/null 2>&1; then
                                echo "Actualizando imagen del deployment..."
                                kubectl -n lab-uac set image deployments/backend-nest-test-uac backend-nest-test-uac=${dockerImagePrefix}/backend-nest-test-uac:${BUILD_NUMBER}
                            else
                                echo "ERROR: El deployment 'backend-nest-test-uac' no existe en el namespace 'lab-uac'."
                                exit 1
                            fi
                        """
                    }
                }
            }
        }
    }
}
