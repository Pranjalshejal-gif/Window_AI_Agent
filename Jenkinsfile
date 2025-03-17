pipeline {
    agent { label 'TehPC' } // Ensure it runs on a Windows agent

    environment {
        GIT_REPO = 'https://github.com/Pranjalshejal-gif/Window_AI_Agent.git'
        FLASK_PORT = '5000'
    }

    parameters {
        string(name: 'TEST_TOPIC', defaultValue: '', description: 'Enter the test topic (optional if using PDF)')
        string(name: 'NUM_CASES', defaultValue: '5', description: 'Enter the number of test cases')
        string(name: 'CSV_FILENAME', defaultValue: 'test_cases', description: 'Enter the CSV filename')
        string(name: 'PDF_FILE_PATH', defaultValue: '', description: 'Enter the absolute path of the PDF file (optional)')
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Set Up Environment') {
            steps {
                bat '''
                    python -m venv venv
                    call venv\\Scripts\\activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pymupdf flask requests google-generativeai
                '''
            }
        }

        stage('Check PDF File') {
            steps {
                script {
                    if (params.PDF_FILE_PATH) {
                        if (!fileExists(params.PDF_FILE_PATH)) {
                            error "❌ ERROR: PDF file not found at: ${params.PDF_FILE_PATH}"
                        }
                        echo "📄 PDF file found: ${params.PDF_FILE_PATH}"
                        env.UPLOADED_PDF = params.PDF_FILE_PATH
                    } else {
                        echo "⚡ No PDF provided, generating test cases from text."
                    }
                }
            }
        }

        stage('Start Flask Server') {
            steps {
                script {
                    echo "🚀 Starting Flask application..."
                    bat 'start /B python app.py > flask_output.log 2>&1'
                    sleep 10

                    def flaskRunning = "down"
                    for (int i = 0; i < 5; i++) {
                        flaskRunning = bat(script: "curl -s http://127.0.0.1:%FLASK_PORT%/health || echo 'down'", returnStdout: true).trim()
                        if (flaskRunning != "down") {
                            break
                        }
                        sleep 3
                    }

                    if (flaskRunning == "down") {
                        error "🔥 ERROR: Flask server failed to start!"
                    }
                    echo "✅ Flask application started successfully!"
                }
            }
        }

        stage('Generate Test Cases') {
            steps {
                script {
                    def jsonResponse = ""

                    if (env.UPLOADED_PDF) {
                        echo "📄 Processing PDF file: ${env.UPLOADED_PDF}"
                        jsonResponse = bat(script: """
                            curl -s -X POST http://127.0.0.1:%FLASK_PORT%/generate_pdf ^
                            -F "pdf_path=${env.UPLOADED_PDF}" ^
                            -F "prompt=${params.TEST_TOPIC}" ^
                            -F "num_cases=${params.NUM_CASES}" ^
                            -F "filename=${params.CSV_FILENAME}"
                        """, returnStdout: true).trim()
                    } else {
                        echo "📝 Generating test cases from text..."
                        jsonResponse = bat(script: """
                            curl -s -X POST http://127.0.0.1:%FLASK_PORT%/generate ^
                            -H "Content-Type: application/json" ^
                            -d "{\\"topic\\": \\"${params.TEST_TOPIC}\\", \\"num_cases\\": ${params.NUM_CASES}, \\"filename\\": \\"${params.CSV_FILENAME}\\"}"
                        """, returnStdout: true).trim()
                    }

                    echo "🔹 API Response: ${jsonResponse}"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline executed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for issues.'
        }
    }
}
