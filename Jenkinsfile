// Jenkinsfile (v15 - 修正 chown 命令，使用正确的用户组)
pipeline {
    agent any

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' // 明确指定 builder 用户所属的组

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 路径变量 ---
        CACHE_DIR = "${env.WORKSPACE}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        HOME_DIR = "${env.WORKSPACE}/.jenkins-home"
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
                        "-u root -v ${env.WORKSPACE}:/project -w /project"
                    ) {
                        sh """
                            set -e
                            
                            echo "========================================================"
                            echo "Step 1: Preparing environment as root..."
                            
                            echo "Creating build user '${BUILD_USER_NAME}' with ID ${BUILD_USER_ID}..."
                            adduser \\
                                --uid ${BUILD_USER_ID} \\
                                --shell /bin/sh \\
                                --ingroup ${BUILD_GROUP_NAME} \\
                                --disabled-password \\
                                --no-create-home \\
                                ${BUILD_USER_NAME}
                            
                            echo "Creating required directories..."
                            mkdir -p /project/.cache
                            mkdir -p /project/.jenkins-home
                            
                            # [核心修改] 将 chown 的组从 'builder' 改为 'root'
                            # 因为 adduser 命令将 builder 用户的主组设置为了 root
                            echo "Setting ownership for the new user..."
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} /project
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user '${BUILD_USER_NAME}' for secure build..."
                            
                            su -s /bin/sh ${BUILD_USER_NAME} -c " \\
                                set -e ; \\
                                echo '--> Now running as: \$(whoami) (ID: \$(id -u))' ; \\
                                echo '--> Checking for cached Godot export templates...' ; \\
                                if [ ! -f '/project/.cache/${TEMPLATE_FILENAME}' ]; then \\
                                    echo '--> Template not found. Downloading...' ; \\
                                    wget -O '/project/.cache/${TEMPLATE_FILENAME}' '${TEMPLATE_URL}' ; \\
                                    echo '--> Download complete.' ; \\
                                else \\
                                    echo '--> Template found in cache. Skipping download.' ; \\
                                fi ; \\
                                echo '--> Installing export templates...' ; \\
                                godot --headless --install-export-templates '/project/.cache/${TEMPLATE_FILENAME}' --user-path /project/.jenkins-home ; \\
                                echo '--> Starting Godot export...' ; \\
                                godot --headless --export-release \\"${EXPORT_PRESET}\\" --user-path /project/.jenkins-home ; \\
                                echo '--> Build completed successfully!' ; \\
                            "
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
