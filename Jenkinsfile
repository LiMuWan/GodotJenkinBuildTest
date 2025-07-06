// Jenkinsfile (v17 - 强制 cd 到项目目录并修正日志输出)
pipeline {
    agent any

    environment {
        // --- 构建配置 ---
        PROJECT_ROOT_IN_CONTAINER = '/project'
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
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

        HOME_DIR = "${PROJECT_ROOT_IN_CONTAINER}/.jenkins-home"
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
                            mkdir -p ${CACHE_DIR}
                            mkdir -p ${HOME_DIR}
                            
                            echo "Setting ownership for the new user..."
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user '${BUILD_USER_NAME}' for secure build..."
                            
                            # [核心修改]
                            # 1. cd ${PROJECT_ROOT_IN_CONTAINER}: 确保 Godot 在正确的项目目录中执行。
                            # 2. export HOME=${PROJECT_ROOT_IN_CONTAINER}: 解决 Godot 写入家目录的问题。
                            # 3. \\\\$(whoami): 双重转义，确保 echo 命令能正确显示切换后的用户名。
                            su -s /bin/sh ${BUILD_USER_NAME} -c " \\
                                cd ${PROJECT_ROOT_IN_CONTAINER} && \\
                                export HOME=${PROJECT_ROOT_IN_CONTAINER} && \\
                                set -e ; \\
                                echo '--> Now running as: \\\\$(whoami) (ID: \\\\$(id -u)) in PWD=\\\\$(pwd)' ; \\
                                echo '--> Checking for cached Godot export templates...' ; \\
                                if [ ! -f '${TEMPLATE_LOCAL_PATH}' ]; then \\
                                    echo '--> Template not found. Downloading...' ; \\
                                    wget -O '${TEMPLATE_LOCAL_PATH}' '${TEMPLATE_URL}' ; \\
                                    echo '--> Download complete.' ; \\
                                else \\
                                    echo '--> Template found in cache. Skipping download.' ; \\
                                fi ; \\
                                echo '--> Installing export templates...' ; \\
                                godot --headless --install-export-templates '${TEMPLATE_LOCAL_PATH}' --user-path ${HOME_DIR} ; \\
                                echo '--> Starting Godot export...' ; \\
                                godot --headless --export-release \\"${EXPORT_PRESET}\\" --user-path ${HOME_DIR} ; \\
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
