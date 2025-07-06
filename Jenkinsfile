// Jenkinsfile (v33 - 安装 xauth 并再次确认目录)
pipeline {
    agent any

    environment {
        // --- 构建配置 ---
        PROJECT_ROOT_IN_CONTAINER = '/project'
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 路径变量 ---
        CACHE_DIR = "${PROJECT_ROOT_IN_CONTAINER}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        // Godot 的用户数据目录
        GODOT_USER_PATH = "${PROJECT_ROOT_IN_CONTAINER}/.godot"
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')

                    godotImage.inside(
                        // -w /project 确保容器启动后的默认目录是 /project
                        "-u root -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            
                            # ========================================================
                            # 阶段一：以 root 用户准备环境
                            # ========================================================
                            echo "Stage 1: Preparing environment as root..."
                            
                            echo "--> Installing dependencies (including xauth)..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb \\
                                xauth \\
                                libxcursor1 \\
                                libxkbcommon0 \\
                                libxinerama1 \\
                                libxi6 \\
                                libdbus-1-3 \\
                                ca-certificates \\
                                wget
                            
                            echo "--> Creating build user '${BUILD_USER_NAME}'..."
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists, skipping creation."
                            
                            echo "--> Creating and setting permissions for required directories..."
                            mkdir -p ${CACHE_DIR}
                            mkdir -p ${GODOT_USER_PATH}
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            # ========================================================
                            # 阶段二：切换到 builder 用户执行构建
                            # ========================================================
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                
                                # 强制切换到项目目录，确保万无一失
                                cd ${PROJECT_ROOT_IN_CONTAINER}
                                
                                echo "--> Now running as: \$(whoami) (ID: \$(id -u))"
                                echo "--> Current directory: \$(pwd)"
                                
                                export HOME=${PROJECT_ROOT_IN_CONTAINER}

                                echo "--> [CRITICAL] Pre-warming Godot inside Xvfb..."
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot --editor --quit --path . --user-path ${GODOT_USER_PATH}
                                echo "--> Pre-warming complete."
                                
                                echo "--> Checking for cached Godot export templates..."
                                if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                    echo "--> Template not found. Downloading..."
                                    wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                    echo "--> Download complete."
                                else
                                    echo "--> Template found in cache."
                                fi
                                
                                echo "--> Installing export templates inside Xvfb..."
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot --verbose --install-export-templates "${TEMPLATE_LOCAL_PATH}" --user-path ${GODOT_USER_PATH} --quit
                                
                                echo "--> Starting Godot export inside Xvfb..."
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot --verbose --export-release "${EXPORT_PRESET}" --path . --user-path ${GODOT_USER_PATH} --quit
                                
                                echo "--> Build completed successfully!"
                            '
                        """
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
