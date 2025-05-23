pipeline {
     agent any

    environment {
        IMAGE_FILE = ''
    }

    stages {
        stage('Setup Environment') {
            steps {
                sh '''
                    apt-get update && apt-get install -y \
                        python3 \
                        python3-pip \
                        qemu-system-arm \
                        wget \
                        unzip \
                        tmux

                    apt-get install -y firefox-esr

                    wget https://github.com/mozilla/geckodriver/releases/download/v0.34.0/geckodriver-v0.34.0-linux64.tar.gz
                    tar -xvzf geckodriver-v0.34.0-linux64.tar.gz
                    chmod +x geckodriver  
                    mv geckodriver /usr/local/bin/          

                    pip3 install pytest requests selenium locust robotframework webdriver-manager --break-system-packages
                '''
            }
        }

        stage('Download OpenBMC Image') {
            steps {
                sh '''
                    wget https://jenkins.openbmc.org/job/ci-openbmc/lastSuccessfulBuild/distro=ubuntu,label=docker-builder,target=romulus/artifact/openbmc/build/tmp/deploy/images/romulus/*zip*/romulus.zip
                    unzip romulus.zip
                '''
            }
        }

        stage('Run OpenBMC in QEMU') {
            steps {
                sh '''
                    tmux new-session -d -s openbmc 'IMAGE_FILE=$(find romulus/ -name "obmc-phosphor-image-romulus-*.static.mtd" -print -quit) ; qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file=$IMAGE_FILE,format=raw,if=mtd -net nic -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu'

                    sleep 180
                '''
            }
        }

        stage('API tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''
                   pytest tests/api/test_redfish.py -v \
                     -k "not power_management" \
                     --junitxml=api_results.xml \
                     --capture=tee-sys \
                     --log-file=openbmc_tests.log \
                     --log-file-level=INFO
                '''
                }
            }
            post {
                always {
                    junit 'api_results.xml'
                    archiveArtifacts artifacts: 'openbmc_tests.log', fingerprint: true
                }
            }
        }

        stage('UI tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''
                   pytest tests/ui/openbmc_ui_tests.py -v \
                     --capture=tee-sys \
                     --log-file=ui_tests.log \
                     --log-file-level=INFO
                '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'ui_tests.log', fingerprint: true
                }
            }
        }

        stage('Load tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                script {
                def exitCode = sh(
                    script: '''

                    timeout 5m locust \
                    -f tests/load/locustfile.py \
                    --headless \
                    --autostart \
                    --users 200 \
                    --spawn-rate 20 \
                    --run-time 4m \
                    --stop-timeout 30 \
                    --csv=locust_report \
                    --csv-full-history \
                    --logfile=load_tests.log \
                    --loglevel INFO \
                    --exit-code-on-error 0

                    ''',
                    returnStatus: true
                )
                
                if (exitCode != 0 && exitCode != 124) {
                    error "Тесты упали с кодом ${exitCode}"
                }

                }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'load_tests.log, locust_report*.csv', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            sh '''
                tmux kill-session -t openbmc 2>/dev/null || true
            '''

            cleanWs()
        }
    }
}
