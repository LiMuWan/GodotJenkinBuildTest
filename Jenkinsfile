// Jenkinsfile (v31 - 修正用户切换逻辑，采用两阶段 inside)
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

                    // =================================================================
                    // 阶段一：以 root 用户身份准备环境
                    // =================================================================
                    godotImage.inside(
                        "-u root -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            echo "========================================================"
                            echo "Stage 1: Preparing environment as root..."
                            
                            echo "--> Installing dependencies..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb \\
                                libxcursor1 \\
                                libxkbcommon0 \\
                                libxinerama1 \\
                                libxi6 \\
                                libdbus-1-3 \\
                                ca-certificates \\
                                wget
                            
                            echo "--> Creating build user '${BUILD_USER_NAME}'..."
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME}
                            
                            echo "--> Creating and setting permissions for required directories..."
                            mkdir -p ${CACHE_DIR}
                            mkdir -p ${GODOT_USER_PATH}
                            # 将整个项目目录的所有权交给 builder 用户
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            echo "--> Environment preparation complete."
                        """
                    }

                    // =================================================================
                    // 阶段二：以 builder 用户身份执行构建
                    // =================================================================
                    godotImage.inside(
                        // 直接指定用户，不再需要 su
                        "-u ${BUILD_USER_ID} -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            echo "========================================================"
                            echo "Stage 2: Running build as user: \$(whoami) (ID: \$(id -u))"
                            echo "--> Current directory: \$(pwd)"
                            
                            # 设置 HOME 环境变量，让 Godot 知道在哪里找配置文件
                            export HOME=${PROJECT_ROOT_IN_CONTAINER}

                            echo '--> [CRITICAL] Pre-warming Godot inside Xvfb...'
                            xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --editor --quit --path . --user-path ${GODOT_USER_PATH}
                            echo '--> Pre-warming complete.'
                            
                            echo '--> Checking for cached Godot export templates...'
                            if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                echo '--> Template not found. Downloading...'
                                wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                echo '--> Download complete.'
                            else
                                echo '--> Template found in cache.'
                            fi
                            
                            echo '--> Installing export templates inside Xvfb...'
                            xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --install-export-templates "${TEMPLATE_LOCAL_PATH}" --user-path ${GODOT_USER_PATH} --quit
                            
                            echo '--> Starting Godot export inside Xvfb...'
                            # 注意：这里 EXPORT_PRESET 不需要额外的引号了，因为整个脚本块已经是字符串
                            xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --export-release "${EXPORT_PRESET}" --path . --user-path ${GODOT_USER_PATH} --quit
                            
                            echo '--> Build completed successfully!'
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
