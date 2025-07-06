// Jenkinsfile (适用于Linux Docker Agent)
pipeline {
    // 指定在一个 Linux Docker 容器内运行。
    agent {
        docker {
            // 使用我们下载的、基于Linux的Godot无头版镜像
            image 'parsenoire/godot-headless:4.4'
            // 关键参数：
            // -v "${pwd()}:/project"  : 将Jenkins当前工作区(pwd)挂载到容器内的 /project 目录
            // -w /project              : 将容器的工作目录设置为 /project，后续命令都在这里执行
            args '-v "${pwd()}:/project" -w /project'
        }
    }

    // 定义环境变量
    environment {
        // 导出预设的名称，与 export_presets.cfg 中的 name="Windows Desktop" 完全对应
        EXPORT_PRESET = 'Windows Desktop'
        // 定义构建输出目录，方便后续归档
        BUILD_OUTPUT_DIR = 'Build/Windows'
    }

    stages {
        // 阶段一：检出代码
        // Jenkins已经在使用Git Parameter插件时完成了代码的拉取，
        // 这个阶段确保工作区是最新的。
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // 阶段二：为 Windows 构建项目
        stage('Build for Windows') {
            steps {
                // 使用 sh 命令，因为我们的构建环境是 Linux 容器
                sh '''
                    echo "========================================================"
                    echo "Starting Godot export for preset: ${EXPORT_PRESET}"
                    echo "Project source is in: $(pwd)"
                    echo "========================================================"

                    # 在Linux容器中，可执行文件是 godot，而不是 godot.exe
                    # Godot 会读取项目中的 export_presets.cfg 并根据预设名称进行导出
                    # 输出路径由 export_presets.cfg 定义，会生成在当前目录(即/project)下
                    godot --headless --export-release "${EXPORT_PRESET}"

                    echo "Build completed!"
                '''
            }
        }

        // 阶段三 (新增)：修复文件权限
        // 这是一个非常重要的步骤，防止工作区清理失败
        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts..."
                    # 将容器内root用户创建的所有文件，所有权交还给Jenkins运行用户
                    chown -R $(id -u):$(id -g) .
                '''
            }
        }

        // 阶段四：归档构建好的程序
        stage('Archive Executable') {
            steps {
                // 使用我们定义的环境变量来归档构建产物
                // **请确保这个路径与您 export_presets.cfg 中定义的输出路径一致！**
                archiveArtifacts artifacts: "${BUILD_OUTPUT_DIR}/**", followSymlinks: false
            }
        }
    }
}
