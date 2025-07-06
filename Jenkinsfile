// Jenkinsfile (适用于Linux Docker Agent - 已修正上下文问题)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // [主要修改] 使用 Jenkins 内建变量 ${env.WORKSPACE} 代替 pwd()。
            // ${env.WORKSPACE} 会在 agent 分配后被正确解析为工作区绝对路径。
            args "-v ${env.WORKSPACE}:/project -w /project"
        }
    }

    environment {
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = 'Build/Windows'
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
                    echo "Starting Godot export for preset: ${EXPORT_PRESET}"
                    echo "Project source is in: $(pwd)"
                    echo "========================================================"

                    godot --headless --export-release "${EXPORT_PRESET}"

                    echo "Build completed!"
                '''
            }
        }

        stage('Fix Permissions') {
            steps {
                sh '''
                    echo "Changing ownership of build artifacts..."
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
