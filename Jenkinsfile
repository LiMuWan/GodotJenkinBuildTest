// Jenkinsfile (v37 - Docker in Docker 正确姿势)
pipeline {
    agent any

    environment {
        // !!! 关键修改：容器内的项目路径就是 Jenkins 的工作区路径 !!!
        PROJECT_ROOT_IN_CONTAINER = "${env.WORKSPACE}" 
        
        EXPORT_PRESET = 'Windows Desktop'
        BUILD_OUTPUT_DIR = "${env.WORKSPACE}/Build/Windows" // 使用绝对路径
        EXPORT_FILENAME = 'GodotJenkinBuildTest.exe'

        BUILD_USER_ID = '1000'
        BUILD_USER_NAME = 'builder'
        BUILD_GROUP_NAME = 'root' 

        // --- 模板配置 ---
        GODOT_RELEASE_TAG = '4.4.1-stable'
        TEMPLATE_FILENAME = "Godot_v${GODOT_RELEASE_TAG}_export_templates.tpz"
        
        // --- 路径变量 ---
        // 全部使用绝对路径，确保万无一失
        CACHE_DIR = "${env.WORKSPACE}/.cache" 
        TEMPLATE_LOCAL_PATH = "${CACHE_DIR}/${TEMPLATE_FILENAME}"
        TEMPLATE_URL = "https://github.com/godotengine/godot/releases/download/${GODOT_RELEASE_TAG}/${TEMPLATE_FILENAME}"
        GODOT_USER_PATH = "${env.WORKSPACE}/.godot"
    }

    stages {
        stage('Build in Docker Container') {
            steps {
                script {
                    def godotImage = docker.image('parsenoire/godot-headless:4.4')

                    // !!! 关键修改：移除 -v 参数，只保留 -u 和 -w !!!
                    // Jenkins会自动处理 --volumes-from，将工作区挂载到容器内相同路径
                    godotImage.inside(
                        "-u root -w '${PROJECT_ROOT_IN_CONTAINER}'"
                    ) {
                        sh """
                            set -e
                            
                            echo "Stage 1: Preparing environment as root..."
                            apt-get update -y && apt-get install -y --no-install-recommends \\
                                xvfb xauth libxcursor1 libxkbcommon0 libxinerama1 \\
                                libxi6 libdbus-1-3 ca-certificates wget libasound2 libpulse0
                            
                            adduser --uid ${BUILD_USER_ID} --shell /bin/sh --ingroup ${BUILD_GROUP_NAME} --disabled-password --no-create-home ${BUILD_USER_NAME} || echo "User '${BUILD_USER_NAME}' already exists."
                            
                            mkdir -p '${CACHE_DIR}' '${GODOT_USER_PATH}'
                            # 更改所有权时，要确保操作的是容器内的路径
                            chown -R ${BUILD_USER_NAME}:${BUILD_GROUP_NAME} '${PROJECT_ROOT_IN_CONTAINER}'
                            
                            echo "Stage 2: Switching to user '${BUILD_USER_NAME}' to run build..."
                            su -s /bin/sh ${BUILD_USER_NAME} -c '
                                set -e
                                # 确保切换到正确的工作目录
                                cd "${PROJECT_ROOT_IN_CONTAINER}"
                                
                                echo "--> Now running as: \$(whoami) in \$(pwd)"
                                
                                # 再次检查文件是否存在，这次必须成功！
                                echo "--- DEBUG: Listing project root contents (ls -la) ---"
                                ls -la

                                echo "--> Preparing output directory..."
                                mkdir -p "${BUILD_OUTPUT_DIR}"
                                
                                echo "--> Starting Godot export..."
                                xvfb-run --auto-servernum --server-args="-screen 0 1280x720x24" godot \\
                                    --verbose \\
                                    --path . \\
                                    --export-release "${EXPORT_PRESET}" \\
                                    "${BUILD_OUTPUT_DIR}/${EXPORT_FILENAME}" \\
                                    --user-path "${GODOT_USER_PATH}" \\
                                    --quit
                                
                                echo "--> Build completed successfully!"
                            '
                        """
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                // 归档路径也需要是相对于 Jenkins 工作区的
                archiveArtifacts artifacts: "Build/Windows/**", followSymlinks: false
            }
        }
    }
}
