// Jenkinsfile (v12 - 使用 docker.image.inside 实现精细控制)
pipeline {
    // 1. 使用 agent any，让 Jenkins 先随便找一个执行器来运行 Pipeline 的框架
    agent any

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 路径变量 ---
        // 注意：因为我们手动挂载，所以这里直接用 Jenkins 的 WORKSPACE 变量
        CACHE_DIR = "${env.WORKSPACE}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        HOME_DIR = "${env.WORKSPACE}/.jenkins-home"
    }

    stages {
        stage('Checkout Code') {
            steps {
                // 清理工作区，确保每次都是干净的构建环境
                cleanWs()
                // 拉取代码
                checkout scm
            }
        }

        stage('Build in Docker Container') {
            steps {
                // 2. 使用 script 块来执行 Scripted Pipeline 语法
                script {
                    // 定义要使用的 Docker 镜像
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')

                    // 3. 使用 inside 块，这是核心
                    // 我们在这里可以传递完整的、自定义的 docker run 参数
                    godotImage.inside(
                        // [核心修改]
                        // -u root: 强制以 root 用户运行，覆盖 Jenkins 全局设置
                        // -v ...: 将宿主机的 WORKSPACE 挂载到容器的 /project
                        // -w /project: 将容器的工作目录设置为 /project
                        "-u root -v ${env.WORKSPACE}:/project -w /project"
                    ) {
                        // --- 现在，我们已经在容器内部，并且是 root 用户 ---
                        
                        sh '''
                            set -e
                            
                            echo "========================================================"
                            echo "Step 1: Preparing directories and permissions..."
                            
                            # 以 root 身份创建所有需要的目录
                            # 注意：容器内的路径是 /project/..., 对应宿主机的 WORKSPACE/...
                            mkdir -p /project/.cache
                            mkdir -p /project/.jenkins-home
                            
                            # 将整个工作区的所有权交给用户 1000，为后续步骤做准备
                            chown -R 1000:1000 /project
                            echo "Directories created and permissions set for user 1000."
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user 1000 for secure build..."
                            
                            # 使用 su 切换到低权限用户 1000 执行所有 Godot 相关命令
                            su -s /bin/sh 1000 -c " \\
                                set -e ; \\
                                echo '--> Now running as user $(id -u)' ; \\
                                echo '--> Checking for cached Godot export templates...' ; \\
                                if [ ! -f '/project/.cache/${TEMPLATE_FILENAME}' ]; then \\
                                    echo '--> Template not found. Downloading...' ; \\
                                    wget -O '/project/.cache/${TEMPLATE_FILENAME}' '${TEMPLATE_URL}' ; \\
                                    echo '--> Download complete.' ; \\
                                else \\
                                    echo '--> Template found in cache. Skipping download.' ; \\
                                fi ; \\
                                echo '--> Installing export templates...' ; \\
                                godot --headless --install-export-templates '/project/.cache/${TEMPLATE_FILENAME}' --user-path /project/.jenkins-home ; \\
                                echo '--> Starting Godot export...' ; \\
                                godot --headless --export-release \\"${EXPORT_PRESET}\\" --user-path /project/.jenkins-home ; \\
                                echo '--> Build completed successfully!' ; \\
                            "
                        '''
                    } // --- 结束 inside 块 ---
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                // Jenkins (在容器外) 需要权限来归档文件，
                // 此时文件所有者可能是容器内的 root 或 1000。
                // 最好是确保 Jenkins 能读取它们。
                // 如果归档失败，可以取消下面这行注释，在容器外执行权限修复。
                // sh "sudo chown -R ${env.USER}:${env.USER} ${env.WORKSPACE}"
                
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
