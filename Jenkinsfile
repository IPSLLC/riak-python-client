pipeline {
    agent {
        label "${env.NODE_LABEL}"
    }

    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Guard') {
            agent {
                label "built-in"
            }

            steps {
                script {
                    if (env.GUARD_GROOVY_CODE) evaluate(env.GUARD_GROOVY_CODE)
                }
            }
        }

        stage('Main') {
            environment {
                MAKE_ARGS = "-j16"
            }

            stages {
                stage('Checkout') {
                    steps {
                        checkout scm
                    }
                }

                stage('Prepare metadata') {
                    steps {
                        script {
                            env.PKG_VERSION = sh(
                                script: "git describe --tags \$(git rev-list --tags --max-count=1)",
                                returnStdout: true
                            ).trim()

                            env.SOURCE_NAME = sh(
                                script: "awk '/^Source:/ {print \$2}' debian/control",
                                returnStdout: true
                            ).trim()
                        }
                    }
                }

                stage('Source package') {
                    steps {
                        sh '''
                            set -eu

                            dch -b -v "${PKG_VERSION}.${BUILD_NUMBER}" \\"---new build---\\"

                            mkdir -p tmp_src
                            cd tmp_src
                            dpkg-source -Zgzip -I -Itmp_src -b ../
                        '''
                    }
                }

                stage('Publish source package') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'apt.registries.creds',
                                usernameVariable: 'USERNAME',
                                passwordVariable: 'PASSWORD'
                            )
                        ]) {
                            sh '''
                                set -eu

                                cd tmp_src

                                for file in *; do
                                    curl -fsSL -u "$USERNAME:$PASSWORD" \
                                        -X POST -F "file=@${file}" \
                                        "$APTLY_API_URL/files/upload_${SOURCE_NAME}"
                                    rm "$file"
                                done

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X POST \
                                    "$APTLY_API_URL/repos/$APT_REPOSITORY_NAME/file/upload_${SOURCE_NAME}"

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X PUT -H 'Content-Type: application/json' \
                                    --data '{}' \
                                    "$APTLY_API_URL/publish/:./$APT_PREFIX"

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X POST \
                                    "$NEXUS_API_URL/repositories/$NEXUS_REPOSITORY_NAME/invalidate-cache"
                            '''
                        }
                    }
                }

                stage('Build package') {
                    steps {
                        sh '''
                            set -eu

                            # Retry apt update
                            for i in 1 2 3; do
                                apt-get update && break || sleep 5
                            done

                            apt-get build-dep -y "$SOURCE_NAME"
                            apt-get source "$SOURCE_NAME"
                            apt-get -y --new-pkgs upgrade

                            cd "${SOURCE_NAME}-${PKG_VERSION}.${BUILD_NUMBER}"

                            dpkg-buildpackage -us -uc -b -rfakeroot
                        '''
                    }
                }

                stage('Publish package') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'apt.registries.creds',
                                usernameVariable: 'USERNAME',
                                passwordVariable: 'PASSWORD'
                            )
                        ]) {
                            sh '''
                                set -eu

                                for file in *.deb; do
                                    [ -e "$file" ] || continue

                                    curl -fsSL -u "$USERNAME:$PASSWORD" \
                                        -X POST -F "file=@${file}" \
                                        "$APTLY_API_URL/files/upload_${SOURCE_NAME}"
                                done

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X POST \
                                    "$APTLY_API_URL/repos/$APT_REPOSITORY_NAME/file/upload_${SOURCE_NAME}"

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X PUT -H 'Content-Type: application/json' \
                                    --data '{}' \
                                    "$APTLY_API_URL/publish/:./$APT_PREFIX"

                                curl -fsSL -u "$USERNAME:$PASSWORD" \
                                    -X POST \
                                    "$NEXUS_API_URL/repositories/$NEXUS_REPOSITORY_NAME/invalidate-cache"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.deb', fingerprint: true
        }

        fixed {
            emailext(
                to: emailextrecipients([requestor(), culprits()]),
                subject: "Build is back to normal: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Check console output at ${env.BUILD_URL}"
            )
        }

        failure {
            emailext(
                to: emailextrecipients([requestor(), culprits()]),
                subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Check console output at ${env.BUILD_URL}

                CONSOLE LOG:
                \${BUILD_LOG}""",
                mimeType: 'text/plain'
            )
        }

        unsuccessful {
            script {
                if (env.UNSUCCESSFUL_GROOVY_CODE) evaluate(env.UNSUCCESSFUL_GROOVY_CODE)
            }
        }
    }
}
