// Jenkinsfile (v42 - 手动解压模板)
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
        // 定义Godot期望的模板目录路径
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
                            
                            # 创建一个有 HOME 目录的用户
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            # ========================================================
                            # ===         终 极 解 决 方 案：手 动 安 装 模 板         ===
                            # ========================================================
                            echo "--> Manually creating Godot template directory..."
                            mkdir -p "${GODOT_TEMPLATE_DIR}"
                            
                            echo "--> Checking for cached Godot export templates..."
                            if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                                echo "--> Template not found. Downloading..."
                                wget -q --show-progress -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                            else
                                echo "--> Template found in cache."
                            fi
                            
                            echo "--> Unzipping templates directly to the target directory..."
                            unzip -o "${TEMPLATE_LOCAL_PATH}" -d "${GODOT_TEMPLATE_DIR}"
                            # 重命名解压出的文件夹，使其内容直接位于版本号目录下
                            mv "${GODOT_TEMPLATE_DIR}/templates/"* "${GODOT_TEMPLATE_DIR}/"
                            rmdir "${GODOT_TEMPLATE_DIR}/templates"

                            echo "--> Setting correct permissions..."
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} "/home/${BUILD_USER_NAME}"
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            # ========================================================

                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                # HOME 目录现在是 /home/builder，是存在的且有权限的
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--> Now running as: \$(whoami) in \$(pwd) with HOME=\${HOME}"
                                
                                echo "--- DEBUG: Verifying template installation... ---"
                                ls -laR "${HOME}/.local/share/godot/export_templates"
                                
                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                # 不再需要任何特殊的 HOME 设置或 --export-templates 参数
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
