pipeline {
    agent any
    parameters {
        string(name: 'TESTER', defaultValue: 'liwt', description: '测试人员名称')
        string(name: 'CHIP', defaultValue: 'nvidia-h100', description: '芯片平台名称')
        choice(name: 'INFRA', choices: ['vllm', 'sglang'], description: '推理框架')
        choice(name: 'PD', choices: ['agg', 'disagg'], description: 'PD分离模式,agg 表示非 PD 分离, disagg 表示 PD 分离')
        string(name: 'MODEL', defaultValue: 'kimi-k2.5', description: '模型名称（served-model-name）')
        string(name: 'MODEL_PATH', defaultValue: '/dingofs/data1/userdata/llms/moonshotai/Kimi-K2.6', description: '模型文件本地路径')
        string(name: 'BASE_URL', defaultValue: 'http://10.201.149.10:8080', description: 'API 地址')
        string(name: 'NUM_CLIENTS', defaultValue: '10', description: '并发客户端数量')
        string(name: 'MAX_ACTIVE_CONVERSATIONS', defaultValue: '30', description: '最大活跃对话数')
        string(name: 'INPUT_FILE', defaultValue: 'generate_multi_turn.json', description: '输入配置文件名')
        text(name: 'RECIPIENTS', defaultValue: 'liwt@zetyun.com', description: '邮件接收者（逗号分隔）')
        string(name: 'WORK_DIR', defaultValue: '/dingofs/data1/userdata/liwt/maas-image/multi-turn-test', description: '测试仓库目录，请不要修改')
    }
    environment {
        SSH_CREDENTIALS = 'HOST_SSH_KEY'
        REMOTE_HOST = '10.201.132.50'
        REMOTE_USER = 'root'
        VLLM_IMAGE = 'vllm/vllm-openai:v0.21.0-cu129'
        REPORTS_DIR = '/dingofs/data1/userdata/liwt/maas-image/multi-turn-test/reports'
    }

    stages {
        stage('环境检查') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    script {
                        println("=== 环境检查 ===")
                        sh """
ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} << 'ENDSSH'
cd ${params.WORK_DIR}
echo "=== 工作目录 ==="
pwd
ls -la
echo "=== 测试参数信息 ==="
echo "测试人员: ${params.TESTER}"
echo "芯片类型: ${params.CHIP}"
echo "模型服务名称: ${params.MODEL}"
echo "模型路径: ${params.MODEL_PATH}"
echo "BASE_URL: ${params.BASE_URL}"
echo "并发客户端数: ${params.NUM_CLIENTS}"
echo "最大活跃对话数: ${params.MAX_ACTIVE_CONVERSATIONS}"
echo "输入配置文件: ${params.INPUT_FILE}"
echo "BUILD_NUMBER: ${BUILD_NUMBER}"
echo "=== Docker 检查 ==="
docker --version
echo "=== 检查配置文件 ==="
cat ${params.INPUT_FILE}
ENDSSH
"""
                    }
                }
            }
        }

        stage('启动容器并运行测试') {
            steps {
                script {
                    def containerName = "multi-turn-test-${params.CHIP}-${params.MODEL}-${BUILD_NUMBER}"
                    def curDateTime = new Date().format('yyyyMMddHHmmss')
                    def reportsDir = "${env.REPORTS_DIR}/${params.TESTER}/${BUILD_NUMBER}/${params.CHIP}/${params.MODEL}/${curDateTime}"
                    env.CONTAINER_NAME = containerName
                    env.CUR_DATE_TIME = curDateTime
                    env.REPORTS_DIR_PATH = reportsDir

                    println("=== 启动 vLLM 容器 ===")
                    println("镜像: ${env.VLLM_IMAGE}")
                    println("容器名: ${containerName}")
                    println("报告输出目录: ${reportsDir}")

                    sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh """
ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} << 'ENDSSH'
set -e
echo "=== 清理旧容器 ==="
docker rm -f ${containerName} 2>/dev/null || true

echo "=== 创建报告目录 ==="
mkdir -p ${reportsDir}

echo "=== 启动容器 ==="
docker run -d --name ${containerName} \
    --network host \
    --memory=32g \
    --shm-size=1g \
    --entrypoint bash \
    -v ${params.WORK_DIR}:/workspace/multi-turn-test \
    -v ${params.MODEL_PATH}:${params.MODEL_PATH} \
    -v ${reportsDir}:${reportsDir} \
    -w /workspace/multi-turn-test \
    ${env.VLLM_IMAGE} \
    -c "sleep infinity"

echo "=== 等待容器启动 ==="
sleep 20

echo "=== 容器状态 ==="
docker ps | grep ${containerName}

echo "=== 安装测试依赖 ==="
export https_proxy=http://100.64.1.68:1080
export http_proxy=http://100.64.1.68:1080
docker exec ${containerName} bash -c "cd /workspace/multi-turn-test && pip install -q -r requirements.txt || pip3 install -q -r requirements.txt"
unset https_proxy
unset http_proxy

echo "=== 检查 Python 命令 ==="
PYTHON_CMD=\$(docker exec ${containerName} bash -c "command -v python3 || command -v python || echo 'python3'")
echo "使用 Python 命令: \${PYTHON_CMD}"
ENDSSH
"""
                            
                            def outputFile = "${reportsDir}/output.json"
                            def statsFile = "${reportsDir}/stats.json"
                            def logFile = "${reportsDir}/test_${BUILD_NUMBER}.log"

                            sh """
ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} << 'ENDSSH'
echo "=== 执行多轮对话测试 ==="
PYTHON_CMD=\$(docker exec ${containerName} bash -c "command -v python3 || command -v python || echo 'python3'")

echo "=== 检查 API 连通性 ==="
HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" ${params.BASE_URL}/v1/models)
if [ "\${HTTP_CODE}" != "200" ]; then
    echo "ERROR: API 连通性检查失败, HTTP状态码: \${HTTP_CODE}, URL: ${params.BASE_URL}/v1/models"
    exit 1
fi
echo "API 连通性检查通过, HTTP状态码: \${HTTP_CODE}"

docker exec ${containerName} bash -c \\
    "cd /workspace/multi-turn-test && \\
    \${PYTHON_CMD} benchmark_serving_multi_turn.py \\
        --model ${params.MODEL_PATH} \\
        --served-model-name ${params.MODEL} \\
        --url '${params.BASE_URL}' \\
        --input-file ${params.INPUT_FILE} \\
        --output-file ${outputFile} \\
        --num-clients ${params.NUM_CLIENTS} \\
        --max-active-conversations ${params.MAX_ACTIVE_CONVERSATIONS} \\
        --trust-remote-code" 2>&1 | tee ${logFile}
echo "=== 测试执行完成 ==="
echo "输出文件: ${outputFile}"
echo "统计文件: ${statsFile}"
echo "日志文件: ${logFile}"
ENDSSH
"""
                        }
                    }
                }
            }
        }

        stage('拉取测试结果') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        script {
                            def curDateTime = env.CUR_DATE_TIME
                            def buildsDir = "builds/${BUILD_NUMBER}"
                            
                            println("=== 拉取测试结果到 Jenkins workspace ===")
                            
                            sh """
set -e
echo "=== 拉取报告目录到 Jenkins workspace ==="
rm -rf ${buildsDir}
mkdir -p builds

scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -r ${REMOTE_USER}@${REMOTE_HOST}:${env.REPORTS_DIR}/${params.TESTER}/${BUILD_NUMBER}/ ./builds/

echo "=== 拉取完成，查看目录结构 ==="
find ./${buildsDir} -type f | head -50
echo "=== 统计文件数量 ==="
find ./${buildsDir} -name '*.json' | wc -l
echo "=== 日志文件数量 ==="
find ./${buildsDir} -name '*.log' | wc -l
"""
                        }
                    }
                }
            }
        }

        stage('发送邮件') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        def buildsDir = "builds/${BUILD_NUMBER}"
                        def curDateTime = env.CUR_DATE_TIME
                        def testStatus = "成功"
                        
                        def logFiles = findFiles(glob: "${buildsDir}/**/*.log")
                        def statsFiles = findFiles(glob: "${buildsDir}/**/stats.json")
                        def outputFiles = findFiles(glob: "${buildsDir}/**/output.json")
                        
                        def parametersSection = ""
                        def statisticsSummary = ""
                        
                        if (logFiles && logFiles.length > 0) {
                            def logContent = readFile(logFiles[0].path)
                            def lines = logContent.split('\n')
                            def inParameters = false
                            def inSummary = false
                            def parametersLines = []
                            def summaryLines = []
                            def paramStarted = false
                            def summaryStarted = false
                            def paramHeaderFound = false
                            
                            for (def line : lines) {
                                if (line.contains("Parameters:")) {
                                    inParameters = true
                                    paramStarted = true
                                    paramHeaderFound = true
                                    parametersLines.add(line)
                                } else if (line.contains("Statistics summary:") || line.contains("Statistics summary")) {
                                    inParameters = false
                                    inSummary = true
                                    summaryStarted = true
                                    summaryLines.add(line)
                                } else if (inParameters && paramHeaderFound) {
                                    if (line.trim().startsWith("---") && parametersLines.size() > 1) {
                                        break
                                    }
                                    parametersLines.add(line)
                                } else if (inSummary && summaryStarted) {
                                    if (line.trim().startsWith("ttft_ms") || 
                                        line.trim().startsWith("tpot_ms") ||
                                        line.trim().startsWith("latency_ms") ||
                                        line.trim().startsWith("input_num_turns") ||
                                        line.trim().startsWith("input_num_tokens") ||
                                        line.trim().startsWith("output_num_tokens") ||
                                        line.trim().startsWith("output_num_chunks")) {
                                        summaryLines.add(line)
                                    }
                                }
                            }
                            
                            parametersSection = parametersLines.join('\n')
                            statisticsSummary = summaryLines.join('\n')
                        }
                        
                        if (!statisticsSummary || statisticsSummary.trim().isEmpty()) {
                            statisticsSummary = "无统计数据"
                            testStatus = "失败/无结果"
                        }
                        if (!parametersSection || parametersSection.trim().isEmpty()) {
                            parametersSection = "无参数信息"
                        }

                        def emailBody = """
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background-color: #f5f5f5; }
        .container { max-width: 1000px; margin: 0 auto; background-color: #fff; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .header { background-color: ${testStatus == '成功' ? '#2196F3' : '#f44336'}; color: white; padding: 20px; border-radius: 5px 5px 0 0; }
        .content { padding: 20px; }
        table { border-collapse: collapse; width: 100%; margin-top: 15px; }
        th, td { border: 1px solid #ddd; padding: 10px; text-align: left; }
        th { background-color: #f2f2f2; }
        h3 { color: #333; border-bottom: 2px solid #2196F3; padding-bottom: 5px; }
        pre { background:#f4f4f4;border:1px solid #ddd;border-radius:4px;padding:12px;overflow-x:auto;margin:10px 0;font-family:monospace;font-size:12px;white-space:pre-wrap;word-break:break-all; }
    </style>
</head>
<body>
<div class="container">
<div class="header">
    <h2 style="margin: 0;">模型推理 - Multi-Turn 对话测试报告 - 构建 #${BUILD_NUMBER}</h2>
</div>
<div class="content">
    <h3>测试概要</h3>
    <table>
        <tr><th>项目</th><td>值</td></tr>
        <tr><td>构建编号</td><td>#${BUILD_NUMBER}</td></tr>
        <tr><td>芯片平台</td><td>${params.CHIP}</td></tr>
        <tr><td>模型名称</td><td>${params.MODEL}</td></tr>
        <tr><td>模型路径</td><td>${params.MODEL_PATH}</td></tr>
        <tr><td>API 地址</td><td>${params.BASE_URL}</td></tr>
        <tr><td>并发客户端数</td><td>${params.NUM_CLIENTS}</td></tr>
        <tr><td>最大活跃对话数</td><td>${params.MAX_ACTIVE_CONVERSATIONS}</td></tr>
        <tr><td>输入配置文件</td><td>${params.INPUT_FILE}</td></tr>
        <tr><td>测试人员</td><td>${params.TESTER}</td></tr>
        <tr><td>测试日期</td><td>${curDateTime}</td></tr>
        <tr><td>执行时间</td><td>${currentBuild.durationString}</td></tr>
        <tr><td>测试状态</td><td>${testStatus}</td></tr>
    </table>

    <h3>Parameters</h3>
    <pre>${parametersSection}</pre>

    <h3>Statistics Summary</h3>
    <pre>${statisticsSummary}</pre>

    <p style="margin-top: 20px;"><b>详细日志和报告已归档到 Jenkins 构建 artifacts 中。</b></p>
    <p>Jenkins 构建地址: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
</div>
<div class="footer" style="margin-top: 20px; padding: 15px; background-color: #f9f9f9;">
    此邮件由 Jenkins 自动发送，请勿回复。
</div>
</div>
</body>
</html>"""

                        println("=== 发送邮件 ===")
                        println("日志文件: ${logFiles ? logFiles.length : 0}")
                        println("统计文件: ${statsFiles ? statsFiles.length : 0}")
                        println("测试状态: ${testStatus}")

                        def attachmentPattern = "builds/${BUILD_NUMBER}/**/*.log,builds/${BUILD_NUMBER}/**/stats.json,builds/${BUILD_NUMBER}/**/output.json"
                        
                        emailext(
                            subject: "[Multi-Turn 测试报告] #${BUILD_NUMBER} ${params.CHIP} - ${params.MODEL}",
                            body: emailBody,
                            to: "${params.RECIPIENTS}",
                            mimeType: 'text/html',
                            attachmentsPattern: attachmentPattern
                        )
                    }
                }
            }
        }

        stage('清理容器') {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                    script {
                        def containerName = env.CONTAINER_NAME ?: "multi-turn-test-${params.CHIP}-${params.MODEL}-${BUILD_NUMBER}"
                        println("=== 清理容器: ${containerName} ===")

                        sh """
ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} << 'ENDSSH'
docker rm -f ${containerName} 2>/dev/null || true
echo "容器 ${containerName} 已删除"
docker ps | grep ${containerName} || echo "容器已清理完成"
ENDSSH
"""
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                archiveArtifacts artifacts: "builds/${BUILD_NUMBER}/**", allowEmptyArchive: true, fingerprint: true
                println("构建完成: ${currentBuild.currentResult}")
            }
        }
        cleanup {
            cleanWs()
        }
    }
}