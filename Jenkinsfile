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
        stage('Example') {
            steps {
                echo 'Hello!'
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
                sh 'mkdir -p results/'
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    echo "ls .zap directory:"
                    ls -l ${WORKSPACE}/.zap
                    ls -l ${WORKSPACE}/.zap
                    echo "Zawartość passive.yaml:"
                    cat ${WORKSPACE}/.zap/passive.yaml || echo "Brak pliku passive.yaml!"
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v ${WORKSPACE}/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c "zap.sh -cmd -addonupdate && zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta && zap.sh -cmd -autorun /zap/wrk/.zap/passive.yaml" || true
                ''' 
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
        }
    }
}

