// Jenkinsfile (最终确认版 - 完整路径逻辑)
pipeline {
    agent any

    environment {
        // --- 构建配置 ---
        // 这是项目在 Docker 容器内的映射路径
        PROJECT_ROOT_IN_CONTAINER = '/project'
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 路径变量 (这些路径都是基于容器内部的 /project) ---
        CACHE_DIR = "${PROJECT_ROOT_IN_CONTAINER}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        // Godot 的用户数据目录，也设置在容器内的项目目录下，以保持隔离
        GODOT_USER_PATH = "${PROJECT_ROOT_IN_CONTAINER}/.godot"
    }

    stages {
        stage('Checkout Code') {
            steps {
                // 清理 Jenkins 的 workspace
                cleanWs()
                // 将代码拉取到 Jenkins 的 workspace
                checkout scm
            }
        }

        stage('Build in Docker Container') {
            steps {
                script {
                    // 指定要使用的 Docker 镜像
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')

                    // 启动容器，并设置关键的映射关系
                    godotImage.inside(
                        // -u root: 以 root 用户身份启动容器，以便安装软件
                        // -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER}: 将 Jenkins workspace 映射到容器的 /project
                        // -w ${PROJECT_ROOT_IN_CONTAINER}: 将容器的默认工作目录设置为 /project
                        "-u root -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            
                            echo "========================================================"
                            echo "Step 1: Preparing environment as root..."
                            
                            echo "--> Installing Xvfb and all required libraries..."
                            apt-get update -y && apt-get install -y \\
                                xvfb \\
                                libxcursor1 \\
                                libxkbcommon0 \\
                                libxinerama1 \\
                                libxi6 \\
                                libdbus-1-3
                            
                            echo "--> Creating build user..."
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME}
                            mkdir -p ${CACHE_DIR}
                            mkdir -p ${GODOT_USER_PATH}
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user '${BUILD_USER_NAME}' for secure build..."
                            
                            # 使用 su 切换到 builder 用户执行所有 Godot 相关命令
                            # -c "..." 中的所有命令都在一个新的 shell 中执行
                            su -s /bin/sh ${BUILD_USER_NAME} -c " \\
                                set -e && \\
                                # 确认当前目录是 /project，即使 -w 已经设置了，这也是一个好习惯
                                cd ${PROJECT_ROOT_IN_CONTAINER} && \\
                                export HOME=${PROJECT_ROOT_IN_CONTAINER} && \\
                                echo '--> Now running as: \$(whoami) (ID: \$(id -u)) in PWD=\$(pwd)' && \\
                                
                                echo '--> [CRITICAL] Pre-warming Godot inside Xvfb...' && \\
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --editor --quit --path . --user-path ${GODOT_USER_PATH} && \\
                                echo '--> Pre-warming complete. All settings files should now exist.' && \\
                                
                                echo '--> Checking for cached Godot export templates...' && \\
                                if [ ! -f ${TEMPLATE_LOCAL_PATH} ]; then \\
                                    echo '--> Template not found. Downloading...' ; \\
                                    wget -q --show-progress -O ${TEMPLATE_LOCAL_PATH} '${TEMPLATE_URL}' ; \\
                                    echo '--> Download complete.' ; \\
                                else \\
                                    echo '--> Template found in cache. Skipping download.' ; \\
                                fi && \\
                                
                                echo '--> Installing export templates inside Xvfb...' && \\
                                # 安装模板不需要 --path 参数
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --install-export-templates ${TEMPLATE_LOCAL_PATH} --user-path ${GODOT_USER_PATH} --quit && \\
                                
                                echo '--> Starting Godot export inside Xvfb...' && \\
                                # 导出命令必须指定项目路径 --path . (代表当前目录)
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --export-release \\"${EXPORT_PRESET}\\" --path . --user-path ${GODOT_USER_PATH} --quit && \\
                                
                                echo '--> Build completed successfully!' \\
                            "
                        """
                    } 
                }
            }
        }

        // 当以上所有步骤成功后，才会执行此阶段
        stage('Archive Artifacts') {
            steps {
                // 将 Jenkins workspace 中（也就是容器 /project 中）的 Build/Windows 目录下的所有文件归档
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
