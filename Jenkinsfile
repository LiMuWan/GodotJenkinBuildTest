// Jenkinsfile
pipeline {
    // 指定在一个 Windows Docker 容器内运行。
    // !!! 重要: 请将 'your-windows-godot-image:latest' 替换为您自己的、
    // 包含 Godot 引擎的 Windows Docker 镜像名称。
    agent {
        docker { image 'godot4-omnibuilder3d:latest-4.4.1' }
    }

    // 定义环境变量
    environment {
        // 导出预设的名称，与 export_presets.cfg 中的 name="Windows Desktop" 完全对应
        EXPORT_PRESET = 'Windows Desktop'
    }

    stages {
        // 阶段一：检出代码
        // Jenkins 会自动将 Git 仓库的所有文件（包括模板）拉取到工作区
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // 阶段二：为 Windows 构建项目
        stage('Build for Windows') {
            steps {
                // 使用 bat 命令，因为我们的目标是 Windows
                // 注意：因为您的 export_presets.cfg 中已经定义了输出路径，
                // 所以我们在这里无需再次指定，Godot 会自动使用它。
                bat '''
                    echo "Running Godot export for preset: %EXPORT_PRESET%"
                    godot.exe --headless --export-release "%EXPORT_PRESET%"
                    echo "Build completed!"
                '''
            }
        }

        // 阶段三：归档构建好的程序
        stage('Archive Executable') {
            steps {
                // 将您在 export_presets.cfg 中定义的 Build/Windows 目录下的所有文件归档
                archiveArtifacts artifacts: 'Build/Windows/**', followSymlinks: false
            }
        }
    }
}
