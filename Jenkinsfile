// Jenkinsfile (v35 - 添加文件系统检查)
pipeline {
    agent any

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
                        "-u root -v ${env.WORKSPACE}:${PROJECT_ROOT_IN_CONTAINER} -w ${PROJECT_ROOT_IN_CONTAINER}"
                    ) {
                        sh """
                            set -e
                            
                            # 阶段一：以 root 用户准备环境
                            echo "Stage 1: Preparing environment as root..."
                            echo "--> Installing dependencies (including audio libs)..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb xauth libxcursor1 libxkbcommon0 libxinerama1 \\
                                libxi6 libdbus-1-3 ca-certificates wget \\
                                libasound2 libpulse0
                            
                            echo "--> Creating build user '${BUILD_USER_NAME}'..."
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            echo "--> Creating and setting permissions for required directories..."
                            mkdir -p ${CACHE_DIR} ${GODOT_USER_PATH}
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            # 阶段二：切换到 builder 用户执行构建
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                cd ${PROJECT_ROOT_IN_CONTAINER}
                                
                                echo "--> Now running as: \$(whoami) (ID: \$(id -u)) in \$(pwd)"
                                export HOME=${PROJECT_ROOT_IN_CONTAINER}

                                # ... 省略之前的成功步骤 ...
                                
                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"

                                # ========================================================
                                # ===           决 定 性 的 调 试 步 骤           ===
                                # ========================================================
                                echo "--- DEBUG: Listing project root contents (ls -la) ---"
                                ls -la
                                
                                echo "--- DEBUG: Displaying export_presets.cfg (cat) ---"
                                # 如果文件不存在，这个命令会报错，这本身也是一个有用的信息
                                cat export_presets.cfg || echo "export_presets.cfg not found."
                                
                                echo "--- DEBUG: End of debug section ---"
                                # ========================================================

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
