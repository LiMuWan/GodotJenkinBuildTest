// Jenkinsfile (v50 - 引入持久化缓存，避免重复安装)
pipeline {
    agent any

    environment {
        // --- 项目与构建配置 ---
        PROJECT_ROOT_IN_CONTAINER = "${env.WORKSPACE}" 
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = "${env.WORKSPACE}/Build/Windows"
        EXPORT_FILENAME = 'GodotJenkinBuildTest.exe'
        
        // --- 用户与权限配置 ---
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 
        
        // --- Godot 版本与模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 缓存相关路径配置 ---
        // 1. 在Jenkins主机上的持久化缓存路径 (按版本隔离，非常好的实践)
        HOST_CACHE_DIR_TEMPLATES = "/var/jenkins_home/cache/godot_templates/${GODOT_RELEASE_TAG}"
        // 2. 模板下载文件的缓存路径
        HOST_CACHE_DIR_DOWNLOADS = "/var/jenkins_home/cache/godot_downloads"
        TEMPLATE_LOCAL_PATH = "${HOST_CACHE_DIR_DOWNLOADS}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
        // 3. 容器内模板的目标安装路径 (用于检查和挂载)
        TEMPLATE_TARGET_DIR_IN_CONTAINER = "/home/${BUILD_USER_NAME}/.local/share/godot/export_templates/${GODOT_RELEASE_TAG}"
    }

    stages {
        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')
                    // 核心改动：使用 -v 参数挂载主机目录到容器内
                    godotImage.inside(
                        """
                        -u root
                        -w '${PROJECT_ROOT_IN_CONTAINER}'
                        -v '${HOST_CACHE_DIR_TEMPLATES}':'/home/${BUILD_USER_NAME}/.local/share/godot/export_templates'
                        -v '${HOST_CACHE_DIR_DOWNLOADS}':'${HOST_CACHE_DIR_DOWNLOADS}'
                        """
                    ) {
                        sh """
                            set -ex  # 开启-e（错误时退出）和-x（打印执行的命令）

                            echo "Stage 1: Preparing environment as root..."
                            apt-get update -y && apt-get install -y --no-install-recommends wget ca-certificates procps
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            echo "--> Ensuring cache directories exist and have correct permissions..."
                            # 在主机上创建目录，并赋予容器内用户写入权限
                            mkdir -p "${HOST_CACHE_DIR_DOWNLOADS}"
                            mkdir -p "/home/${BUILD_USER_NAME}/.local/share/godot/export_templates"
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "/home/${BUILD_USER_NAME}/.local"
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -ex
                                export HOME="/home/${BUILD_USER_NAME}"
                                
                                # =====================================================================
                                # ===               缓存逻辑：检查模板是否存在              ===
                                # =====================================================================
                                if [ ! -d "${TEMPLATE_TARGET_DIR_IN_CONTAINER}" ]; then
                                    echo "--> Export templates not found in cache. Starting installation..."
                                    
                                    echo "--> Downloading templates if not present in cache..."
                                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                        wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                    else
                                        echo "--> Template .tpz file found in cache. Skipping download."
                                    fi
                                    
                                    cd "\${HOME}" # 进入中立目录
                                    
                                    echo "--> Installing export templates in the background..."
                                    godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}" < /dev/null &
                                    GODOT_PID=\$!

                                    echo "--> Monitoring installation progress (PID: \${GODOT_PID})..."
                                    TIMEOUT=600
                                    COUNT=0
                                    while ! kill -0 \${GODOT_PID} 2>/dev/null; do
                                        # 改进监控逻辑：直接检查进程是否存在
                                        if [ \$COUNT -ge \$TIMEOUT ]; then
                                            echo "!!!--> ERROR: Template installation timed out after \${TIMEOUT} seconds."
                                            exit 1
                                        fi
                                        echo "    ... waiting for installation to complete (elapsed: \${COUNT}s)"
                                        sleep 5
                                        COUNT=\$((COUNT + 5))
                                    done
                                    
                                    echo "--> SUCCESS: Template installation process finished."
                                    # 验证一下最终目录是否真的存在
                                    if [ ! -d "${TEMPLATE_TARGET_DIR_IN_CONTAINER}" ]; then
                                       echo "!!!--> CRITICAL ERROR: Godot process finished but target directory was not created!"
                                       exit 1
                                    fi

                                else
                                    echo "--> SUCCESS: Found cached export templates. Skipping installation."
                                fi
                                # =====================================================================

                                echo "--> Entering project directory to perform build..."
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--- DEBUG: Verifying template installation... ---"
                                ls -laR "\${HOME}/.local/share/godot/export_templates"
                                
                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                godot \\
                                    --headless \\
                                    --verbose \\
                                    --path . \\
                                    --export-release "${EXPORT_PRESET}" \\
                                    "${BUILD_OUTPUT_DIR}/${EXPORT_FILENAME}" \\
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
                archiveArtifacts artifacts: "Build/Windows/**", followSymlinks: false, onlyIfSuccessful: true
            }
        }
    }
    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
