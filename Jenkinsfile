// Jenkinsfile (v25 - 补全更多 X11 依赖)
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

        // Godot 的 --user-path 目录
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
                            
                            echo "========================================================"
                            echo "Step 1: Preparing environment as root..."
                            
                            echo "--> Installing Xvfb and all known required X11 libraries..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb \\
                                libxcursor1 \\
                                libxkbcommon0 \\
                                libxinerama1 \\
                                libxi6
                            
                            echo "--> Creating build user..."
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME}
                            mkdir -p ${CACHE_DIR}
                            mkdir -p ${GODOT_USER_PATH}
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} ${PROJECT_ROOT_IN_CONTAINER}
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user '${BUILD_USER_NAME}' for secure build..."
                            
                            su -s /bin/sh ${BUILD_USER_NAME} -c " \\
                                set -e && \\
                                cd ${PROJECT_ROOT_IN_CONTAINER} && \\
                                export HOME=${PROJECT_ROOT_IN_CONTAINER} && \\
                                echo '--> Now running as: \$(whoami) (ID: \$(id -u)) in PWD=\$(pwd)' && \\
                                
                                echo '--> [CRITICAL] Pre-warming Godot inside Xvfb...' && \\
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --editor --quit --user-path ${GODOT_USER_PATH} && \\
                                echo '--> Pre-warming complete. All settings files should now exist.' && \\
                                
                                echo '--> Checking for cached Godot export templates...' && \\
                                if [ ! -f ${TEMPLATE_LOCAL_PATH} ]; then \\
                                    echo '--> Template not found. Downloading...' ; \\
                                    wget -q --show-progress -O ${TEMPLATE_LOCAL_PATH} '${TEMPLATE_URL}' ; \\
                                    echo '--> Download complete.' ; \\
                                else \\
                                    echo '--> Template found in cache. Skipping download.' ; \\
                                fi && \\
                                
                                echo '--> Installing export templates inside Xvfb...' && \\
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --install-export-templates ${TEMPLATE_LOCAL_PATH} --user-path ${GODOT_USER_PATH} && \\
                                
                                echo '--> Starting Godot export inside Xvfb...' && \\
                                xvfb-run --auto-servernum --server-args='-screen 0 1280x720x24' godot --verbose --export-release \\"${EXPORT_PRESET}\\" --user-path ${GODOT_USER_PATH} && \\
                                
                                echo '--> Build completed successfully!' \\
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
