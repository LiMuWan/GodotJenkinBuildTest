// Jenkinsfile (最终优化版 v3 - 修正模板文件名)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            args "-v ${env.WORKSPACE}:/project -w /project -e HOME=/project/.jenkins-home"
        }
    }

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'

        // --- 模板配置 ---
        // [修改 1] 使用 TEMPLATE_TAG 来匹配 GitHub 的发布标签 (e.g., 4.4.1.stable)
        TEMPLATE_TAG = '4.4.1.stable'
        // [修改 2] 使用 TEMPLATE_FILENAME_VERSION 来匹配实际的文件名 (e.g., 4.4.1-stable)
        TEMPLATE_FILENAME_VERSION = '4.4.1-stable'
        
        // [修改 3] 使用新的变量来构造正确的文件名和 URL
        TEMPLATE_FILENAME = "Godot_v${TEMPLATE_FILENAME_VERSION}_export_templates.tpz"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${TEMPLATE_TAG}/${TEMPLATE_FILENAME}"
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
                        
                        # 使用 wget 下载，这次的 URL 和文件名都是正确的
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
