// Jenkinsfile (v8 - 修正 Shell 脚本注释)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // 将 Jenkins 工作空间挂载到容器的 /project 目录
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
        
        // 将模板的路径设置为工作空间内的 .cache 目录，实现持久化缓存
        CACHE_DIR = '/project/.cache' 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"

        // 定义 HOME 环境变量，供 Godot 在容器内使用
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
                    # 设置脚本在遇到任何错误时立即退出
                    set -e 

                    echo "========================================================"
                    echo "Step 1: Checking for cached Godot export templates..."
                    
                    # 检查前先确保缓存目录存在
                    mkdir -p "${CACHE_DIR}"

                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                        echo "Template not found in cache. Downloading from ${TEMPLATE_URL}..."
                        wget -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                        echo "Download complete. Template is now cached for future builds."
                    else
                        echo "Template found in cache at ${TEMPLATE_LOCAL_PATH}. Skipping download."
                    fi
                    
                    echo "========================================================"
                    echo "Step 2: Preparing Godot user directory at ${HOME}"
                    mkdir -p "${HOME}"
                    echo "Directory is ready."
                    
                    echo "========================================================"
                    echo "Step 3: Installing export templates..."
                    godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}"
                    echo "Templates installed successfully."

                    echo "========================================================"
                    echo "Step 4: Starting Godot export for preset '${EXPORT_PRESET}'"
                    godot --headless --export-release "${EXPORT_PRESET}"
                    echo "Build completed successfully!"
                '''
            }
        }

        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts to match Jenkins user..."
                    # 确保所有在容器内生成的文件（构建产物、缓存、配置）的所有权都正确
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
