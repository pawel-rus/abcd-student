pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-token', url: 'https://github.com/pawel-rus/abcd-student', branch: 'main'
                }
            }
        }

        // Stage to ensure no conflicting containers exist
        stage('Ensure No Conflicting Containers') {
            steps {
                script {
                    // Stopping and removing any existing juice-shop container
                    sh '''
                        docker ps -a -q --filter "name=juice-shop" | xargs --no-run-if-empty docker stop
                        docker ps -a -q --filter "name=juice-shop" | xargs --no-run-if-empty docker rm
                    '''
                    
                    // Stopping and removing any existing zap container
                    sh '''
                        docker ps -a -q --filter "name=zap" | xargs --no-run-if-empty docker stop
                        docker ps -a -q --filter "name=zap" | xargs --no-run-if-empty docker rm
                    '''
                }
            }
        }
        
        stage('[ZAP] passive-scan') {
            steps {
                sh 'mkdir -p ${WORKSPACE}/results/'
                sh 'mkdir -p "${WORKSPACE}/.zap/reports"'
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                script {
                    echo "WORKSPACE path: ${env.WORKSPACE}"
                }
                

                 sh """
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v ${env.WORKSPACE}/.zap/:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c '
                            zap.sh -cmd -addonupdate &&
                            zap.sh -cmd -addoninstall communityScripts &&
                            zap.sh -cmd -addoninstall pscanrulesAlpha &&
                            zap.sh -cmd -addoninstall pscanrulesBeta &&
                            zap.sh -cmd -autorun /zap/wrk/passive.yaml
                        ' || true
                """

                
                // sh '''
                //     echo "Listing contents of .zap directory in Jenkins workspace:"
                //     ls -l ${WORKSPACE}/.zap
                //     echo "Contents of passive.yaml in Jenkins workspace:"
                //     cat ${WORKSPACE}/.zap/passive.yaml || echo "File passive.yaml not found!"
                // '''
                // Uruchomienie ZAP i skopiowanie plików
                // sh '''
                //     echo "Starting ZAP container for scanning..."
                //     docker run -d --name zap \
                //         --add-host=host.docker.internal:host-gateway \
                //         -t ghcr.io/zaproxy/zaproxy:stable 

                //     echo "Creating target directory in ZAP container..."
                //     docker exec zap mkdir -p /zap/wrk/
                //     docker exec zap mkdir -p /zap/wrk/reports

                //     echo "Copying scan configuration into ZAP container..."
                //     docker cp ${WORKSPACE}/.zap/. zap:/zap/wrk/
        
                //     echo "Listing contents of /zap/wrk/.zap inside the ZAP container:"
                //     docker exec zap ls -l /zap/wrk/
        
                    
                //     echo "Running ZAP passive scan with addon installation and autorun..."
                //     docker exec zap bash -c \
                //     "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive_scan.yaml" \
                //     || true

                // '''
            }
        }
    }
    post {
        always {
            sh '''
                docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                docker stop zap juice-shop
                docker rm zap
            '''
            archiveArtifacts artifacts: 'results/zap_*.html, results/zap_*.xml', fingerprint: true

        }
    }
}

