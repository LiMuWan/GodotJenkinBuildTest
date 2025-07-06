// Jenkinsfile (v10 - 修正所有语法上下文的注释)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // 以 root 用户启动容器，以便我们有权限修改目录所有权。
            // 同时将 Jenkins 工作区挂载到容器的 /project 目录。
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
                    # ======================================================================
                    # Stage A: 以 root 用户身份准备环境
                    # ======================================================================
                    
                    # 我们现在是 root 用户，第一步就是把工作目录的所有权交给用户 1000
                    # 这样后续的用户 1000 才能在里面创建文件和目录
                    echo "Taking ownership of the workspace for user 1000..."
                    chown -R 1000:1000 /project

                    # ======================================================================
                    # Stage B: 切换到低权限用户 (1000) 执行所有构建命令
                    # ======================================================================

                    # 使用 su 命令，切换到用户 1000 来执行所有真正的构建命令，这更安全
                    # -s /bin/sh 指定使用 sh shell
                    # 1000 是我们要切换到的用户 ID
                    # -c "..." 里的内容就是要在新 shell 中执行的命令
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

        stage('Fix Permissions') {
            // 这个 stage 现在是可选的，因为前面的 chown 已经处理了主要权限。
            // 但保留它可以作为最后一步保障，确保 Jenkins 能完全控制所有产出物。
            // 注意：这里的注释是在 Groovy 上下文中，所以用 //
            steps {
                sh '''
                    echo "Final permission check to ensure Jenkins ownership..."
                    chown -R $(id -u):$(id -g) .
                '''
            }
        }

        stage('Archive Executable') {
            steps {
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
