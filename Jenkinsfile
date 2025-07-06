// Jenkinsfile (v46 - 分离模板安装与项目构建)
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
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
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
                            apt-get update -y && apt-get install -y --no-install-recommends wget ca-certificates
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            echo "--> Creating cache directory and downloading templates..."
                            mkdir -p "${CACHE_DIR}"
                            if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                            fi

                            echo "--> Setting correct permissions..."
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "/home/${BUILD_USER_NAME}"
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                export HOME="/home/${BUILD_USER_NAME}"
                                
                                # =====================================================================
                                # ===      终 极 解 决 方 案：在 中 立 目 录 下 安 装 模 板      ===
                                # =====================================================================
                                echo "--> Entering HOME directory to perform global setup..."
                                cd "\${HOME}"

                                echo "--> Installing export templates using Godot official command..."
                                godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}" --quit
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
