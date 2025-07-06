// Jenkinsfile (v11 - 使用 'user' 参数强制以 root 身份启动容器)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            
            // [核心修改] 使用专用的 'user' 参数来覆盖 Jenkins 的全局用户设置。
            // 这将确保容器以 root 用户身份启动，以便我们拥有执行 chown 的权限。
            user 'root'
            
            // args 参数仍然需要，用于挂载卷和设置工作目录。
            args "-v ${env.WORKSPACE}:/project -w /project"
        }
    }

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        CACHE_DIR = '/project/.cache' 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        HOME = '/project/.jenkins-home'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build for Windows') {
            steps {
                sh '''
                    # Stage A: 以 root 用户身份准备环境
                    # ----------------------------------------------------------------------
                    # 因为 agent 中设置了 user 'root'，所以下面的命令将由 root 执行
                    
                    echo "Taking ownership of the workspace for user 1000..."
                    chown -R 1000:1000 /project
                    echo "Ownership changed successfully."

                    # Stage B: 切换到低权限用户 (1000) 执行所有构建命令
                    # ----------------------------------------------------------------------
                    # 这是一个好习惯，避免在 root 用户下执行整个构建过程
                    
                    echo "Switching to user 1000 to run the build..."
                    su -s /bin/sh 1000 -c " \\
                        set -e ; \\
                        echo '========================================================' ; \\
                        echo 'Step 1: Checking for cached Godot export templates...' ; \\
                        mkdir -p '${CACHE_DIR}' ; \\
                        if [ ! -f '${TEMPLATE_LOCAL_PATH}' ]; then \\
                            echo 'Template not found in cache. Downloading...' ; \\
                            wget -O '${TEMPLATE_LOCAL_PATH}' '${TEMPLATE_URL}' ; \\
                            echo 'Download complete.' ; \\
                        else \\
                            echo 'Template found in cache. Skipping download.' ; \\
                        fi ; \\
                        echo '========================================================' ; \\
                        echo 'Step 2: Preparing Godot user directory at ${HOME}' ; \\
                        mkdir -p '${HOME}' ; \\
                        echo 'Directory is ready.' ; \\
                        echo '========================================================' ; \\
                        echo 'Step 3: Installing export templates...' ; \\
                        godot --headless --install-export-templates '${TEMPLATE_LOCAL_PATH}' ; \\
                        echo 'Templates installed successfully.' ; \\
                        echo '========================================================' ; \\
                        echo 'Step 4: Starting Godot export for preset \\'${EXPORT_PRESET}\\'...' ; \\
                        godot --headless --export-release '${EXPORT_PRESET}' ; \\
                        echo 'Build completed successfully!' ; \\
                    "
                '''
            }
        }

        stage('Final Permission Fix') {
            // 这个 stage 确保最终 Jenkins 主机上的 jenkins 用户拥有对产出文件的完全控制权
            steps {
                sh '''
                    echo "Final permission check to ensure Jenkins ownership..."
                    # chown 回 Jenkins 运行时的用户 ID 和组 ID
                    chown -R $(id -u):$(id -g) .
                '''
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
