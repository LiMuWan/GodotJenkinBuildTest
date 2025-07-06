// Jenkinsfile (v44 - 使用 unzip -j)
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
        GODOT_TEMPLATE_DIR = "/home/${BUILD_USER_NAME}/.local/share/godot/export_templates/${GODOT_RELEASE_TAG}"
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
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                unzip wget ca-certificates
                            
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            echo "--> Manually creating directories..."
                            mkdir -p "${CACHE_DIR}"
                            mkdir -p "${GODOT_TEMPLATE_DIR}"
                            
                            echo "--> Checking for cached Godot export templates..."
                            if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                echo "--> Template not found. Downloading..."
                                wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                            else
                                echo "--> Template found in cache."
                            fi
                            
                            # ========================================================
                            # ===         终 极 解 决 方 案：unzip -j         ===
                            # ========================================================
                            echo "--> Unzipping templates directly into the target directory, ignoring paths..."
                            # 使用 -j 参数，直接将所有模板文件解压到目标目录，无需再进行 mv 和 rmdir
                            unzip -j -o "${TEMPLATE_LOCAL_PATH}" -d "${GODOT_TEMPLATE_DIR}"
                            # ========================================================

                            echo "--> Setting correct permissions for all build-related directories..."
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "/home/${BUILD_USER_NAME}"
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--> Now running as: \$(whoami) in \$(pwd) with HOME=\${HOME}"
                                
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
