// Jenkinsfile (v6 - 解决权限并采用标准安装模板流程)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // [重要] 将 HOME 环境变量的定义移到 environment 区块，使其在 sh 脚本中也能被直接引用
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
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
        TEMPLATE_LOCAL_PATH = "/tmp/${TEMPLATE_FILENAME}"

        // [重要] 定义 HOME 环境变量，供 Godot 使用
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
                    set -e // 如果任何命令失败，立即终止脚本

                    echo "========================================================"
                    echo "Step 1: Downloading Godot export templates..."
                    
                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                        echo "Template not found locally. Downloading from ${TEMPLATE_URL}..."
                        wget -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                        echo "Download complete."
                    else
                        echo "Template already exists. Skipping download."
                    fi

                    // [核心修改 1] 解决权限问题
                    echo "========================================================"
                    echo "Step 2: Preparing Godot user directory at ${HOME}"
                    mkdir -p "${HOME}"
                    echo "Directory created."
                    
                    // [核心修改 2] 采用标准的“先安装，再导出”流程
                    echo "========================================================"
                    echo "Step 3: Installing export templates..."
                    godot --headless --install-export-templates "${TEMPLATE_LOCAL_PATH}"
                    echo "Templates installed successfully."

                    echo "========================================================"
                    echo "Step 4: Starting Godot export for preset '${EXPORT_PRESET}'"
                    // 现在不需要 --export-templates 参数了，Godot 会自动找到已安装的模板
                    godot --headless --export-release "${EXPORT_PRESET}"
                    echo "Build completed successfully!"
                '''
            }
        }

        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts to match Jenkins user..."
                    // 注意：现在也要包括新创建的 .jenkins-home 目录
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

