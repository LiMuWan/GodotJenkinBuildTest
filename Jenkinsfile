// Jenkinsfile (v52 - 授予对整个HOME目录的完全所有权)
pipeline {
    agent any

    environment {
        // ... (所有环境变量保持不变) ...
        PROJECT_ROOT_IN_CONTAINER = "${env.WORKSPACE}" 
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = "${env.WORKSPACE}/Build/Windows"
        EXPORT_FILENAME = 'GodotJenkinBuildTest.exe'
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        HOST_CACHE_DIR_TEMPLATES = "/var/jenkins_home/cache/godot_templates/${GODOT_RELEASE_TAG}"
        HOST_CACHE_DIR_DOWNLOADS = "/var/jenkins_home/cache/godot_downloads"
        TEMPLATE_LOCAL_PATH = "${HOST_CACHE_DIR_DOWNLOADS}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
        TEMPLATE_TARGET_DIR_IN_CONTAINER = "/home/${BUILD_USER_NAME}/.local/share/godot/export_templates/${GODOT_RELEASE_TAG}"
    }

    stages {
        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')
                    godotImage.inside(
                        """
                        -u root
                        -w '${PROJECT_ROOT_IN_CONTAINER}'
                        -v '${HOST_CACHE_DIR_TEMPLATES}':'/home/${BUILD_USER_NAME}/.local/share/godot/export_templates'
                        -v '${HOST_CACHE_DIR_DOWNLOADS}':'${HOST_CACHE_DIR_DOWNLOADS}'
                        """
                    ) {
                        sh """
                            set -ex

                            echo "Stage 1: Preparing environment as root..."
                            apt-get update -y && apt-get install -y --no-install-recommends wget ca-certificates procps
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            echo "--> Ensuring cache directories exist and have correct permissions..."
                            mkdir -p "${HOST_CACHE_DIR_DOWNLOADS}"
                            # 注意：我们不再需要在这里创建/home/builder/.local...，因为下一步chown会处理整个home
                            
                            # =====================================================================
                            # ===          最终的、根本的、决定性的权限修复          ===
                            # === 将整个 /home/builder 目录的所有权完全移交给 builder  ===
                            # =====================================================================
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "/home/${BUILD_USER_NAME}"
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "${HOST_CACHE_DIR_DOWNLOADS}"
                            # =====================================================================
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -ex
                                export HOME="/home/${BUILD_USER_NAME}"
                                
                                # ... (后续所有脚本逻辑保持不变) ...
                                if [ ! -d "${TEMPLATE_TARGET_DIR_IN_CONTAINER}" ]; then
                                    echo "--> Export templates not found in cache. Starting installation..."
                                    
                                    echo "--> Downloading templates if not present in cache..."
                                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                        wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                    else
                                        echo "--> Template .tpz file found in cache. Skipping download."
                                    fi
                                    
                                    cd "\${HOME}"
                                    
                                    echo "--> Installing export templates in the background..."
                                    godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}" < /dev/null &
                                    GODOT_PID=\$!

                                    echo "--> Monitoring installation progress (PID: \${GODOT_PID})..."
                                    TIMEOUT=600
                                    COUNT=0
                                    while kill -0 \${GODOT_PID} 2>/dev/null; do
                                        if [ \$COUNT -ge \$TIMEOUT ]; then
                                            echo "!!!--> ERROR: Template installation timed out after \${TIMEOUT} seconds."
                                            kill \${GODOT_PID}
                                            exit 1
                                        fi
                                        echo "    ... waiting for installation to complete (elapsed: \${COUNT}s)"
                                        sleep 5
                                        COUNT=\$((COUNT + 5))
                                    done
                                    
                                    echo "--> SUCCESS: Template installation process finished."
                                    if [ ! -d "${TEMPLATE_TARGET_DIR_IN_CONTAINER}" ]; then
                                       echo "!!!--> CRITICAL ERROR: Godot process finished but target directory was not created! Check logs for errors."
                                       exit 1
                                    fi
                                else
                                    echo "--> SUCCESS: Found cached export templates. Skipping installation."
                                fi

                                echo "--> Entering project directory to perform build..."
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--- DEBUG: Verifying template installation... ---"
                                ls -laR "\${HOME}/.local/share/godot/export_templates"
                                
                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                godot \
                                    --headless \
                                    --verbose \
                                    --path . \
                                    --export-release "${EXPORT_PRESET}" \
                                    "${BUILD_OUTPUT_DIR}/${EXPORT_FILENAME}" \
                                    --quit
                                
                                echo "--> Build completed successfully!"
                            '
                        """
                    }
                }
            }
        }
        // ... (后续 stages 保持不变) ...
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
