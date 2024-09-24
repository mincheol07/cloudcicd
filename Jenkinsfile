pipeline {
    agent { label 'master' }

    environment {
        AWS_REGION = 'ap-northeast-2'  // AWS 지역 설정
        ECR_REGISTRY = '339712848231.dkr.ecr.ap-northeast-2.amazonaws.com'  // AWS 계정 ID를 YOUR_ACCOUNT_ID로 대체
        ECR_REPOSITORY_FLASK = 'flaskapp'  // Flask 앱을 위한 ECR 저장소 이름
        ECR_REPOSITORY_NGINX = 'nginx'  // NGINX를 위한 ECR 저장소 이름
        
        FLASK_IMAGE_TAG = "${env.BUILD_ID}-flask"  // 고유한 Flask 이미지 태그
        NGINX_IMAGE_TAG = "${env.BUILD_ID}-nginx"  // 고유한 Nginx 이미지 태그
        GIT_CREDENTIALS_ID = 'github'
        AWS_CREDENTIALS = 'ECR'  // Jenkins AWS 자격 증명 ID
        EKS_CLUSTER_NAME = 'fpekscluster'  // EKS 클러스터 이름 
    }

    stages {
        stage('소스 코드 체크아웃') {
            steps {
                git branch: 'main', url: 'https://github.com/mincheol07/cloudcicd.git'
            }
        }

        stage('ECR 인증') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIALS]]) {
                        sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                        '''
                    }
                }
            }
        }

        stage('Docker 이미지 빌드 FLASKAPP') {
            steps {
                script {
                    // Flask 앱의 Dockerfile은 root 디렉토리에 위치
                    dockerImageFlask = docker.build("${ECR_REGISTRY}/${ECR_REPOSITORY_FLASK}:${FLASK_IMAGE_TAG}", ".")
                }
            }
        }

        stage('Docker 이미지 빌드 NGINX') {
            steps {
                script {
                    // Nginx Dockerfile은 nginx 디렉토리에 위치
                    dockerImageNginx = docker.build("${ECR_REGISTRY}/${ECR_REPOSITORY_NGINX}:${NGINX_IMAGE_TAG}", "nginx")
                }
            }
        }

        stage('ECR에 Docker 이미지 푸시') {
            steps {
                script {
                    dockerImageFlask.push()  // Flask 이미지 ECR로 푸시
                    dockerImageNginx.push()  // Nginx 이미지 ECR로 푸시
                }
            }
        }

        stage('Kubernetes 매니페스트 업데이트') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh '''
                    #!bin/bash
                    cd /home/ec2-user/team3k8s/cloud/
                    git pull origin main --rebase
                    sudo sed -i "s|image: .*/nginx:.*$|image: ${ECR_REGISTRY}/${ECR_REPOSITORY_NGINX}:${NGINX_IMAGE_TAG}|" deployment.yaml  
                    sudo sed -i "s|image: .*/flask:.*$|image: ${ECR_REGISTRY}/${ECR_REPOSITORY_FLASK}:${FLASK_IMAGE_TAG}|" deployment.yaml  
                    git add .
                    git config user.email "chojo480912@gmail.com"
                    git config user.name "mincheol07"
                    git commit -m "Update image tag to ${IMAGE_TAG}"
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mincheol07/team3k8s.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // 워크스페이스 정리
        }
    }
}
