// Jenkinsfile (适用于Linux Docker Agent - 已修正引号问题)
pipeline {
    agent {
        docker {
            image 'parsenoire/godot-headless:4.4'
            // [主要修改] 将此处的单引号改为双引号，以允许Groovy解析 ${pwd()} 变量
            args "-v ${pwd()}:/project -w /project"
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
