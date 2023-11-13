pipeline {

    // 将kubernetes的cloud配置文件化
    agent {
        kubernetes {
            // 新建一个cloud，能连接上k8s即可
            cloud 'k8s'
            label 'jenkins-slave'
            workspaceVolume hostPathWorkspaceVolume(hostPath: "${JENKINS_HOME}")
            serviceAccount 'jenkins'
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                app: jenkins-slave
            spec:
              containers:
              - name: docker
                image: docker
                imagePullPolicy: IfNotPresent
                command:
                - cat
                tty: true
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
                - name: docker-config
                  mountPath: /etc/docker/daemon.json
              - name: curl
                image: curlimages/curl
                imagePullPolicy: IfNotPresent
                command:
                - cat
                tty: true
                volumeMounts:
              securityContext:
                runAsGroup: 0
                runAsUser: 0
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
              - name: docker-config
                hostPath:
                  path: /etc/docker/daemon.json
            '''
        }
    }

    options {
        timestamps()  // 日志添加时间
        disableConcurrentBuilds() // 禁止并行构建
        timeout(time: 1, unit: 'HOURS') // 流水线超时时间
    }

    // 新设置一些环境变量
    environment {
        ALI_IMAGE_REGISTRY = credentials('ali-docker-image-registry')
        HOST_WORKSPACE = "${WORKSPACE}".replaceAll("${AGENT_WORKDIR}", "${JENKINS_HOME}")
    }

    // 构建时选择参数
    parameters {
        string(name: 'CUSTOM_TAG', defaultValue: '', description: '是否自定义tag，为空时使用commitId前5位')
        booleanParam(name: 'UPDATE_DEPLOY', defaultValue: true, description: '是否更新deployment镜像')
    }

    stages {
        stage('环境变量') {
            steps {
                // 在script中给环境变量赋值
                script {
                    sh 'git log --oneline -n 1 > gitlog.file'
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                script {
                    if (env.CUSTOM_TAG == '') {
                        env.IMAGE_TAG = "${GIT_COMMIT}".substring(0, 5)
                    } else {
                        env.IMAGE_TAG = CUSTOM_TAG
                    }
                    env.IMAGE_NAME = "registry.cn-chengdu.aliyuncs.com/nonosword/cicd_demo:${IMAGE_TAG}"
                }
                sh 'printenv'
            }
        }

        stage('拉取源码') {
            steps {
                // 直接从Jenkinsfile所在的scm拉取代码
                checkout scm
            }
        }

        stage('单元测试') {
            steps {
                // 在指定的容器中执行，由kuberntes插件提供
                container('docker') {
                    sh "docker run -v $HOST_WORKSPACE:/app --workdir /app golang:1.17 go test"
                    echo '单元测试通过'
                }
            }
        }

        stage('构建镜像') {
            steps {
                container('docker') {
                    // 如果失败，重试两次
                    retry (2) {
                        // 使用bash执行
                        sh "docker build -t $IMAGE_NAME ."
                        echo "构建镜像成功，信息： $GIT_LOG"
                    }
                }
            }
        }

        stage('上传镜像') {
            steps {
                container('docker') {
                    sh 'docker login -u $ALI_IMAGE_REGISTRY_USR registry.cn-chengdu.aliyuncs.com -p $ALI_IMAGE_REGISTRY_PSW'
                    sh "docker push $IMAGE_NAME"
                    echo '上传镜像完毕'
                }
            }
        }

        stage('更新pod') {
            steps {
                container('curl') {
                    // 步骤的超时时间
                    timeout(time: 1, unit: 'MINUTES') {
                        // 使用该方法来指定shebang
                        sh '''#!/bin/sh
                            cd /var/run/secrets/kubernetes.io/serviceaccount
                            curl https://kubernetes.default/apis/apps/v1/namespaces/default/deployments/cicd-demo \
                            -H "Authorization: Bearer `cat token`" \
                            -H "Content-type:application/strategic-merge-patch+json" \
                            --cacert ca.crt -X PATCH \
                            -d '{"spec": {"template": {"spec": {"containers": [{"name": "cicd-demo","image": "'$IMAGE_NAME'"}]}}}}'
                        '''
                    }
                }
            }
        }
    }

    // 后处理
    post {
        success {
            echo 'success'
        }
        failure {
            echo 'failure'
        }
        always {
            echo 'always'
        }
    }
}