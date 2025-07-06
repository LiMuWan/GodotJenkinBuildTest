// Jenkinsfile (v38 - 修正 HOME 环境变量)
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
        GODOT_USER_PATH = "${env.WORKSPACE}/.godot"
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
                                xvfb xauth libxcursor1 libxkbcommon0 libxinerama1 \\
                                libxi6 libdbus-1-3 ca-certificates wget libasound2 libpulse0
                            
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            mkdir -p '${CACHE_DIR}' '${GODOT_USER_PATH}'
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            # !!! 终极修正：在 su 命令中直接设置 HOME 环境变量 !!!
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                export HOME="${PROJECT_ROOT_IN_CONTAINER}" # 确保 HOME 指向工作区
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--> Now running as: \$(whoami) in \$(pwd) with HOME=\${HOME}"
                                
                                # 模板下载和安装步骤需要在这里执行，以确保在正确的用户和HOME环境下
                                echo "--> Checking for cached Godot export templates..."
                                if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                    echo "--> Template not found. Downloading..."
                                    wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                                    echo "--> Download complete."
                                else
                                    echo "--> Template found in cache."
                                fi
                                
                                echo "--> Installing export templates..."
                                # Godot 会自动使用 HOME/.local/share/godot/export_templates/
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}" --quit

                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                # 移除 --user-path，让 Godot 使用默认路径（现在是正确的 HOME 路径）
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot \\
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
                archiveArtifacts artifacts: "Build/Windows/**", followSymlinks: false
            }
        }
    }
}

