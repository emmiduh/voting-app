pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.7-openjdk-18-slim
    command: ['sleep']
    args: ['infinity']
  - name: nodejs
    image: node:24.5.0
    command: ['sleep']
    args: ['infinity']
  - name: python
    image: python:3.11-slim
    command: ['sleep']
    args: ['infinity']
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ['sleep']
    args: ['infinity']
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep']
    args: ['infinity']
'''
        }
    }
    
    stages {
        // ==========================================
        // WORKER APP STAGES
        // ==========================================
        stage("Worker - Build") {
            when { changeset "**/worker/**" }
            steps {
                container('maven') {
                    echo 'Compiling worker app..'
                    dir('worker') { sh 'mvn compile' }
                }
            }
        }
        stage("Worker - Test") {
            when { changeset "**/worker/**" }
            steps {
                container('maven') {
                    echo 'Running unit tests on worker app..'
                    dir('worker') { sh 'mvn clean test' }
                }
            }
        }
        stage("Worker - Package JAR") {
            when { changeset "**/worker/**" }
            steps {
                container('maven') {
                    echo 'Packaging worker app into a jarfile'
                    dir('worker') {
                        sh 'mvn package -DskipTests'
                        archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    }
                }
            }
        }
        stage('Worker - Docker Package') {
            when {
                changeset "**/worker/**"
                branch 'main'
            }
            steps {
                container('kaniko') {
                    echo 'Building and pushing worker image...'
                    withCredentials([usernamePassword(credentialsId: 'dockerlogin', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh '''
                            mkdir -p /kaniko/.docker
                            AUTH=$(echo -n "$DOCKER_USER:$DOCKER_PASS" | base64 | tr -d '\n')
                            echo "{\\"auths\\":{\\"https://index.docker.io/v1/\\":{\\"auth\\":\\"${AUTH}\\"}}}" > /kaniko/.docker/config.json
                            
                            /kaniko/executor --context `pwd`/worker --dockerfile `pwd`/worker/Dockerfile --destination emmiduh93/worker:v${BUILD_ID} --destination emmiduh93/worker:latest
                        '''
                    }
                }
            }
        }

        // ==========================================
        // RESULT APP STAGES
        // ==========================================
        stage('Result - Build') {
            when { changeset "**/result/**" }
            steps {
                container('nodejs') {
                    echo 'Compiling result app..'
                    dir('result') { sh 'npm install' }
                }
            }
        }
        stage('Result - Test') {
            when { changeset '**/result/**' }
            steps {
                container('nodejs') {
                    echo 'Running Unit Tests on result app..'
                    dir('result') {
                        sh '''npm install
                              npm test'''
                    }
                }
            }
        }
        stage('Result - Docker Package') {
            when {
                changeset "**/result/**"
                branch 'main'
            }
            steps {
                container('kaniko') {
                    echo 'Building and pushing result image...'
                    withCredentials([usernamePassword(credentialsId: 'dockerlogin', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh '''
                            mkdir -p /kaniko/.docker
                            AUTH=$(echo -n "$DOCKER_USER:$DOCKER_PASS" | base64 | tr -d '\n')
                            echo "{\\"auths\\":{\\"https://index.docker.io/v1/\\":{\\"auth\\":\\"${AUTH}\\"}}}" > /kaniko/.docker/config.json
                            
                            /kaniko/executor --context `pwd`/result --dockerfile `pwd`/result/Dockerfile --destination emmiduh93/result:v${BUILD_ID} --destination emmiduh93/result:latest
                        '''
                    }
                }
            }
        }

        // ==========================================
        // VOTE APP STAGES
        // ==========================================
        stage('Vote - Build & Test') { 
            when { changeset "**/vote/**" }
            steps { 
                container('python') {
                    echo 'Compiling and Testing vote app.' 
                    dir('vote') {
                        sh "pip install -r requirements.txt"
                        sh 'nosetests -v'
                    } 
                }
            } 
        } 
        stage('Vote - Docker Package') {
            when {
                changeset "**/vote/**"
                branch 'main'
            }
            steps {
                container('kaniko') {
                    echo 'Building and pushing vote image...'
                    withCredentials([usernamePassword(credentialsId: 'dockerlogin', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh '''
                            mkdir -p /kaniko/.docker
                            AUTH=$(echo -n "$DOCKER_USER:$DOCKER_PASS" | base64 | tr -d '\n')
                            echo "{\\"auths\\":{\\"https://index.docker.io/v1/\\":{\\"auth\\":\\"${AUTH}\\"}}}" > /kaniko/.docker/config.json
                            
                            /kaniko/executor --context `pwd`/vote --dockerfile `pwd`/vote/Dockerfile --destination emmiduh93/vote:v${BUILD_ID} --destination emmiduh93/vote:latest
                        '''
                    }
                }
            }
        }

        // ==========================================
        // DEPLOYMENT STAGE
        // ==========================================
        stage('Deploy to Dev') {
            when {
                // Ensure deployment only happens from the main branch
                branch 'main'
            }
            steps {
                container('kubectl') {
                    echo 'Applying Kubernetes manifests to Dev environment...'
                    sh 'kubectl create namespace voting-app --dry-run=client -o yaml | kubectl apply -f -'
                    sh 'kubectl apply -n voting-app -f k8s-specifications/'
                }
            }
        }
    } 
    
    post {
        always {
            echo 'Monorepo pipeline execution completed.'
        }
    }
}
