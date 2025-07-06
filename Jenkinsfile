// Jenkinsfile (v4 - 修正 URL 路径和文件名)
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
        // [核心修改] 使用一个统一的变量来表示 GitHub 的发布标签，这个标签同时用于 URL 路径和文件名。
        // 这次的值是完全正确的 '4.4.1-stable'
        GODOT_RELEASE_TAG = '4.4.1-stable'
        
        // 使用这个统一的变量来构造正确的文件名和 URL
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
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
                        # 这次的 URL 将会是 100% 正确的
                        echo "Template not found locally. Downloading from ${TEMPLATE_URL}..."
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
