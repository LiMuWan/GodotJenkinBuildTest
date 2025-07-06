// Jenkinsfile (最终优化版 v2 - 使用 wget 代替 curl)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // 同样保留 HOME 环境变量的设置，这是个好习惯
            args "-v ${env.WORKSPACE}:/project -w /project -e HOME=/project/.jenkins-home"
        }
    }

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'

        // --- 模板配置 ---
        TEMPLATE_VERSION = '4.4.1.stable'
        TEMPLATE_FILENAME = "Godot_v${TEMPLATE_VERSION}_export_templates.tpz"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${TEMPLATE_VERSION}/${TEMPLATE_FILENAME}"
        TEMPLATE_LOCAL_PATH = "/tmp/${TEMPLATE_FILENAME}"
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
                    echo "========================================================"
                    echo "Step 1: Checking for Godot export templates..."
                    
                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                        echo "Template not found locally. Downloading from ${TEMPLATE_URL}..."
                        
                        # [主要修改] 使用 wget 命令来下载文件。
                        # -q: 静默模式，减少不必要的输出
                        # -O: 指定输出文件的路径和名称 (注意是大写字母O)
                        wget -q -O "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                        
                        echo "Download complete."
                    else
                        echo "Template already exists at ${TEMPLATE_LOCAL_PATH}. Skipping download."
                    fi

                    echo "========================================================"
                    echo "Step 2: Starting Godot export for preset '${EXPORT_PRESET}'"
                    
                    godot --headless --export-templates "${TEMPLATE_LOCAL_PATH}" --export-release "${EXPORT_PRESET}"

                    echo "Build completed successfully!"
                '''
            }
        }

        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts to match Jenkins user..."
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
