// Jenkinsfile (v13 - 在容器内动态创建构建用户)
pipeline {
    agent any

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
        BUILD_USER_ID = '1000' // 定义构建用户的ID，方便复用

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
                        // --- 我们现在在容器内部，并且是 root 用户 ---
                        
                        sh '''
                            set -e
                            
                            echo "========================================================"
                            echo "Step 1: Preparing environment as root..."
                            
                            # [核心修改] 创建一个用于构建的用户和组
                            # -u ${BUILD_USER_ID}: 指定用户ID
                            # -m: 创建用户的主目录 (虽然我们不用，但这是好习惯)
                            # -s /bin/sh: 指定shell
                            # -N: 不创建同名的私有组，而是使用默认的 users 组
                            echo "Creating build user with ID ${BUILD_USER_ID}..."
                            adduser -u ${BUILD_USER_ID} -m -s /bin/sh -D -G root builder
                            
                            echo "Creating required directories..."
                            mkdir -p /project/.cache
                            mkdir -p /project/.jenkins-home
                            
                            # 将整个工作区的所有权交给新创建的用户
                            echo "Setting ownership for the new user..."
                            chown -R ${BUILD_USER_ID}:${BUILD_USER_ID} /project
                            
                            echo "========================================================"
                            echo "Step 2: Switching to user 'builder' (${BUILD_USER_ID}) for secure build..."
                            
                            # 使用 su 切换到新创建的用户来执行所有 Godot 命令
                            su -s /bin/sh builder -c " \\
                                set -e ; \\
                                echo '--> Now running as: $(whoami) (ID: $(id -u))' ; \\
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
                        '''
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
