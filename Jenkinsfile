pipeline {
    agent any
    
    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }

    parameters {
        gitParameter name: 'branch',
        type: 'PT_BRANCH',
        branchFilter: 'origin/(.*)',
        defaultValue: 'main',
        selectedValue: 'DEFAULT',
        sortMode: 'ASCENDING_SMART',
        description: '选择需要构建的分支'
    }

    stages {
        // =========================================================
        // 1. 【初始化 & 清理】防止磁盘空间不足
        // =========================================================
        stage('初始化 & 磁盘清理') {
            steps {
                cleanWs() 
                script {
                    echo "正在清理悬空镜像以释放磁盘空间..."
                    // 加上 || true 防止如果没有垃圾镜像时报错停止
                    sh 'docker image prune -f || true'
                    sh 'df -h' // 打印磁盘空间供检查
                }
            }
        }

        // =========================================================
        // 2. 【环境预检】提前获取端口，获取失败则立即终止 (Fail Fast)
        // =========================================================
        stage('环境预检 & 端口获取') {
            steps {
                script {
                    echo "正在预检服务配置..."
                    
                    // 修复：使用双引号以便 Jenkins 解析变量 ${JOB_NAME} 和 ${type}
                    def portResult = sh(returnStdout: true, script: "/usr/local/bin/port.sh ${JOB_NAME}-${type}").trim()
                    
                    // 严格校验逻辑：必须获取到端口，否则直接报错退出
                    if (portResult == "" || portResult == null) {
                        error "❌ 构建失败终止：未获取到服务 [${JOB_NAME}-${type}] 的端口配置！请检查 port.sh 脚本。"
                    }
                    
                    // 赋值全局环境变量，后面的阶段都可以直接用
                    echo "✅ 成功获取端口：${portResult}"
                    env.port = portResult
                    env.nodePort = "3" + portResult 
                    
                    echo "预检通过 -> 服务: ${JOB_NAME}-${type}, 内部端口: ${env.port}, NodePort: ${env.nodePort}"
                }
            }
        }

        stage('服务信息')    {
            steps {
                // 修复：使用双引号，否则 $branch 无法解析
                sh "echo 分支：${branch}"
                sh 'echo 构建服务类型：${JOB_NAME}-$type'
            }
        }

        stage('拉取代码') {
            steps {
                checkout([$class: 'GitSCM',
                // 修复：使用双引号 "${branch}"
                branches: [[name: "${branch}"]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'CleanBeforeCheckout']], 
                submoduleCfg: [],
                userRemoteConfigs: [[credentialsId: 'gitlab-cert', url: 'ssh://git@10.250.104.50:2222/root/go-zero-looklook.git']]])
            }
        }

        stage('获取commit_id') {
            steps {
                echo '获取commit_id'
                script {
                    env.commit_id = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                }
            }
        }

        stage('拉取配置文件') {
            steps {
                checkout([$class: 'GitSCM',
                // 修复：使用双引号 "${branch}"
                branches: [[name: "${branch}"]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'conf'], [$class: 'CleanBeforeCheckout']],
                submoduleCfg: [],
                userRemoteConfigs: [[credentialsId: 'gitlab-cert', url: 'ssh://git@10.250.104.50:2222/root/looklook-pro-conf.git']]])
            }
        }

        stage('goctl版本检测') {
            steps{
                sh '/usr/local/bin/goctl -v'
            }
        }

        stage('Dockerfile Build') {
            steps{
                 sh 'yes | cp  -rf conf/${JOB_NAME}/${type}/${JOB_NAME}.yaml  app/${JOB_NAME}/cmd/${type}/etc'
                 // 先删除旧 Dockerfile 再生成
                 sh 'cd app/${JOB_NAME}/cmd/${type} && rm -f Dockerfile && /usr/local/bin/goctl docker -go ${JOB_NAME}.go && ls -l'
                 script{
                     // 生成带构建号的 Image Tag，触发 K8s 滚动更新
                     env.image = sh(returnStdout: true, script: 'echo ${JOB_NAME}-${type}:${commit_id}-${BUILD_NUMBER}').trim()
                 }
                 sh 'echo 镜像名称：${image} && cp app/${JOB_NAME}/cmd/${type}/Dockerfile ./  && ls -l && docker build  -t ${image} .'
            }
        }

        stage('上传到镜像仓库') {
            steps{
                sh 'docker login --username=${docker_username} --password=${docker_pwd} http://${docker_repo}'
                sh 'docker tag  ${image} ${docker_repo}/go-zero-looklook/${image}'
                sh 'docker push ${docker_repo}/go-zero-looklook/${image}'
            }
        }

        stage('部署到k8s') {
            steps{
                script{
                    env.deployYaml = sh(returnStdout: true, script: 'echo ${JOB_NAME}-${type}-deploy.yaml').trim()
                    // 注意：这里不再获取端口，直接使用开头获取到的 env.port 和 env.nodePort
                }

                sh 'echo 正在部署端口：${port}, NodePort: ${nodePort}'
                sh 'rm -f ${deployYaml}'

                // 使用 env.nodePort 和 env.port
                sh '/usr/local/bin/goctl kube deploy -secret docker-login -replicas 2 -nodePort ${nodePort} -requestCpu 200 -requestMem 50 -limitCpu 300 -limitMem 100 -name ${JOB_NAME}-${type} -namespace go-zero-looklook -image ${docker_repo}/go-zero-looklook/${image} -o ${deployYaml} -port ${port} -serviceAccount find-endpoints '
                
                sh 'sed -i "s#autoscaling/v2beta2#autoscaling/v2#g" ${deployYaml}'
                
                sh '/usr/local/bin/kubectl apply -f ${deployYaml}'
            }
        }
    }

    // =========================================================
    // 3. 【构建后清理】无论成功失败都执行
    // =========================================================
    post {
        always {
            script {
                echo "执行构建后清理..."
                // 清理本次构建产生的本地镜像
                if (env.image) {
                    sh 'docker rmi -f ${image} || true'
                    sh 'docker rmi -f ${docker_repo}/go-zero-looklook/${image} || true'
                }
                // 再次尝试清理垃圾，保持服务器干净
                sh 'docker image prune -f || true'
            }
            cleanWs notFailBuild: true
        }
    }
}