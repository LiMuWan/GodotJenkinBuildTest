// Jenkinsfile (v40 - 强制指定模板路径)
pipeline {
    agent any

    environment {
        PROJECT_ROOT_IN_CONTAINER = "${env.WORKSPACE}" 
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = "${env.WORKSPACE}/Build/Windows"
        EXPORT_FILENAME = 'GodotJenkinBuildTest.exe'
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        CACHE_DIR = "${env.WORKSPACE}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
    }

    stages {
        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')
                    godotImage.inside(
                        "-u root -w '${PROJECT_ROOT_IN_CONTAINER}'"
                    ) {
                        sh """
                            set -e
                            
                            echo "Stage 1: Preparing environment as root..."
                            apt-get update -y >/dev/null && apt-get install -y --no-install-recommends >/dev/null \\
                                xvfb xauth libxcursor1 libxkbcommon0 libxinerama1 \\
                                libxi6 libdbus-1-3 ca-certificates wget libasound2 libpulse0
                            
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            mkdir -p '${CACHE_DIR}'
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                export HOME="${PROJECT_ROOT_IN_CONTAINER}"
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--> Now running as: \$(whoami) in \$(pwd) with HOME=\${HOME}"
                                
                                echo "--> Checking for cached Godot export templates..."
                                if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                    echo "--> Template not found. Downloading..."
                                    wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                else
                                    echo "--> Template found in cache."
                                fi
                                
                                echo "--> Installing export templates... (This step is for verification, might be redundant)"
                                # 使用 --headless 确保无GUI交互，即使失败也不影响后续
                                godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}" --quit || true

                                # 修正验证路径
                                echo "--- DEBUG: Verifying template installation directory... ---"
                                ls -laR "${HOME}/.local/share/godot/export_templates" || echo "!!! Template directory not found or is empty !!!"
                                echo "--- DEBUG: End of verification ---"
                                
                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export with explicit template path..."
                                # !!! 终极解决方案：使用 --headless 并强制指定模板路径 !!!
                                godot \\
                                    --headless \\
                                    --verbose \\
                                    --path . \\
                                    --export-release "${EXPORT_PRESET}" \\
                                    "${BUILD_OUTPUT_DIR}/${EXPORT_FILENAME}" \\
                                    --export-templates "${TEMPLATE_LOCAL_PATH}" \\
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
                // 清理旧的构建产物，只归档最新的
                cleanWs deleteDirs: true, patterns: [[pattern: 'Build/**', type: 'INCLUDE']]
                archiveArtifacts artifacts: "Build/Windows/**", followSymlinks: false, onlyIfSuccessful: true
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline successfully completed!'
            // 可以在这里添加发送通知等操作
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
