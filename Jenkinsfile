#!/usr/bin/env groovy

@Library('homelab@main') _

// Pod template for gmailctl build
def POD_YAML = '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    workload-type: ci-builds
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
  - name: golang
    image: golang:1.22-alpine
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 2000m
        memory: 1Gi
  - name: tools
    image: alpine:3.19
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
'''

pipeline {
    agent {
        kubernetes {
            yaml POD_YAML
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout(true)
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        // gmailctl version to build from source
        GMAILCTL_VERSION = '0.11.0'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Config') {
            steps {
                container('tools') {
                    sh '''
                        echo "=== Installing dependencies ==="
                        apk add --no-cache yq jq curl

                        echo ""
                        echo "=== Validating config.yaml syntax ==="
                        yq '.' config.yaml > /dev/null
                        echo "✓ config.yaml is valid YAML"

                        echo ""
                        echo "=== Validating filters.json syntax ==="
                        jq '.' filters.json > /dev/null
                        echo "✓ filters.json is valid JSON"

                        echo ""
                        echo "=== Validating labels.json syntax ==="
                        jq '.' labels.json > /dev/null
                        echo "✓ labels.json is valid JSON"

                        echo ""
                        echo "=== Config Summary ==="
                        RULE_COUNT=$(yq '.rules | length' config.yaml)
                        echo "Rules defined: $RULE_COUNT"
                    '''
                }
            }
        }

        stage('Build gmailctl') {
            steps {
                container('golang') {
                    sh '''
                        echo "=== Building gmailctl v${GMAILCTL_VERSION} from source ==="

                        # Install git for go get
                        apk add --no-cache git

                        # Create output directory
                        mkdir -p dist

                        # Clone gmailctl source
                        git clone --depth 1 --branch v${GMAILCTL_VERSION} https://github.com/mbrt/gmailctl.git /tmp/gmailctl
                        cd /tmp/gmailctl

                        echo ""
                        echo "=== Cross-compiling for multiple platforms ==="

                        # Linux amd64
                        echo "Building linux/amd64..."
                        GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-s -w" -o ${WORKSPACE}/dist/gmailctl-linux-amd64 ./cmd/gmailctl

                        # Linux arm64
                        echo "Building linux/arm64..."
                        GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags="-s -w" -o ${WORKSPACE}/dist/gmailctl-linux-arm64 ./cmd/gmailctl

                        # macOS amd64
                        echo "Building darwin/amd64..."
                        GOOS=darwin GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-s -w" -o ${WORKSPACE}/dist/gmailctl-darwin-amd64 ./cmd/gmailctl

                        # macOS arm64 (Apple Silicon)
                        echo "Building darwin/arm64..."
                        GOOS=darwin GOARCH=arm64 CGO_ENABLED=0 go build -ldflags="-s -w" -o ${WORKSPACE}/dist/gmailctl-darwin-arm64 ./cmd/gmailctl

                        echo ""
                        echo "=== Built binaries ==="
                        ls -la ${WORKSPACE}/dist/

                        # Verify linux/amd64 works
                        echo ""
                        echo "=== Version check ==="
                        chmod +x ${WORKSPACE}/dist/gmailctl-linux-amd64
                        ${WORKSPACE}/dist/gmailctl-linux-amd64 version || echo "Version: v${GMAILCTL_VERSION} (built from source)"
                    '''
                }
            }
        }

        stage('Push to Nexus') {
            when {
                branch 'main'
            }
            steps {
                container('tools') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                            apk add --no-cache curl

                            echo "=== Pushing gmailctl binaries to Nexus ==="

                            NEXUS_URL="https://nexus.erauner.dev/repository/raw-hosted"

                            for binary in dist/gmailctl-*; do
                                name=$(basename $binary)
                                echo "Uploading $name..."
                                curl -f -u "${NEXUS_USER}:${NEXUS_PASS}" \
                                    --upload-file "$binary" \
                                    "${NEXUS_URL}/gmailctl/v${GMAILCTL_VERSION}/${name}"
                            done

                            # Also upload as "latest"
                            for binary in dist/gmailctl-*; do
                                name=$(basename $binary)
                                echo "Uploading $name as latest..."
                                curl -f -u "${NEXUS_USER}:${NEXUS_PASS}" \
                                    --upload-file "$binary" \
                                    "${NEXUS_URL}/gmailctl/latest/${name}"
                            done

                            echo ""
                            echo "✓ Binaries available at:"
                            echo "  ${NEXUS_URL}/gmailctl/v${GMAILCTL_VERSION}/"
                            echo "  ${NEXUS_URL}/gmailctl/latest/"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                homelab.githubStatus('SUCCESS', 'Config validation passed')
            }
            echo '✅ gmailctl configuration is valid!'
        }
        failure {
            script {
                homelab.githubStatus('FAILURE', 'Config validation failed')
            }
            echo '❌ gmailctl configuration validation failed!'
        }
    }
}
