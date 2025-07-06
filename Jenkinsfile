// Jenkinsfile (v36 - 回归初心)
pipeline {
    agent any // 让 Jenkins 在顶层处理 checkout

    environment {
        // --- 构建配置 ---
        PROJECT_ROOT_IN_CONTAINER = '/project'
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
        EXPORT_FILENAME = 'GodotJenkinBuildTest.exe'

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
        GODOT_USER_PATH = "${PROJECT_ROOT_IN_CONTAINER}/.godot"
    }

    stages {
        // !!! 删除 'Checkout Code' 阶段 !!!
        // Jenkins 会在所有 stage 之前，自动在顶层 agent 上执行一次 checkout

        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')

                    godotImage.inside(
                        // -u root 启动，-v 挂载 Jenkins 已检出的工作区
                        "-u root -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            
                            # 阶段一：以 root 用户准备环境
                            echo "Stage 1: Preparing environment as root..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb xauth libxcursor1 libxkbcommon0 libxinerama1 \\
                                libxi6 libdbus-1-3 ca-certificates wget libasound2 libpulse0
                            
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            mkdir -p ${CACHE_DIR} ${GODOT_USER_PATH}
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            # 阶段二：切换到 builder 用户执行构建
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                cd ${PROJECT_ROOT_IN_CONTAINER}
                                
                                export HOME=${PROJECT_ROOT_IN_CONTAINER}

                                # 再次进行决定性的调试检查
                                echo "--- DEBUG: Listing project root contents (ls -la) ---"
                                ls -la

                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot \\
                                    --verbose \\
                                    --path . \\
                                    --export-release "${EXPORT_PRESET}" \\
                                    "${BUILD_OUTPUT_DIR}/${EXPORT_FILENAME}" \\
                                    --user-path ${GODOT_USER_PATH} \\
                                    --quit
                                
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
