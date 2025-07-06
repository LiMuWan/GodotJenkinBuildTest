// Jenkinsfile (最终优化版 - 动态下载模板)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // [修改 1] 新增 -e HOME=... 参数，为容器内的 Godot 进程提供一个可写的“主目录”，
            // 这会解决所有关于 /.local, /.config, /.cache 的权限错误。
            args "-v ${env.WORKSPACE}:/project -w /project -e HOME=/project/.jenkins-home"
        }
    }

    environment {
        // --- 构建配置 ---
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'

        // --- 模板配置 (在这里修改版本即可) ---
        TEMPLATE_VERSION = '4.4.1.stable'
        TEMPLATE_FILENAME = "Godot_v${TEMPLATE_VERSION}_export_templates.tpz"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${TEMPLATE_VERSION}/${TEMPLATE_FILENAME}"
        // 将模板下载到容器的临时目录，这个路径将在构建步骤中使用
        TEMPLATE_LOCAL_PATH = "/tmp/${TEMPLATE_FILENAME}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                // 拉取最新的代码
                checkout scm
            }
        }

        stage('Build for Windows') {
            steps {
                sh '''
                    echo "========================================================"
                    echo "Step 1: Checking for Godot export templates..."
                    
                    # 检查模板文件是否已存在于临时目录，如果不存在，则从网络下载
                    if [ ! -f "${TEMPLATE_LOCAL_PATH}" ]; then
                        echo "Template not found locally. Downloading from ${TEMPLATE_URL}..."
                        # 使用 curl 下载文件。-L 会自动处理重定向，-s 静默模式，-o 指定输出文件
                        curl -sL -o "${TEMPLATE_LOCAL_PATH}" "${TEMPLATE_URL}"
                        echo "Download complete."
                    else
                        echo "Template already exists at ${TEMPLATE_LOCAL_PATH}. Skipping download."
                    fi

                    echo "========================================================"
                    echo "Step 2: Starting Godot export for preset '${EXPORT_PRESET}'"
                    
                    # [修改 2] 在 godot 命令中，使用 --export-templates 参数明确指定我们刚刚下载的模板文件。
                    # Godot 将使用这个 .tpz 文件进行导出，不再去默认目录寻找。
                    godot --headless --export-templates "${TEMPLATE_LOCAL_PATH}" --export-release "${EXPORT_PRESET}"

                    echo "Build completed successfully!"
                '''
            }
        }

        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts to match Jenkins user..."
                    # 修复容器内创建的所有文件（包括构建产物和.jenkins-home）的属主，
                    # 确保 Jenkins 后续可以清理工作区。
                    chown -R $(id -u):$(id -g) .
                '''
            }
        }

        stage('Archive Executable') {
            steps {
                // 将构建产物打包存档，方便在 Jenkins 界面下载
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
