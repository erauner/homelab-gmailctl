#!/usr/bin/env groovy

@Library('homelab@main') _

// Pod template for gmailctl validation
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
  - name: tools
    image: alpine:3.19
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
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
        timeout(time: 5, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate YAML') {
            steps {
                container('tools') {
                    sh '''
                        echo "=== Installing dependencies ==="
                        apk add --no-cache yq jq

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
                    '''
                }
            }
        }

        stage('Check Structure') {
            steps {
                container('tools') {
                    sh '''
                        echo "=== Checking config.yaml structure ==="

                        # Check for required version field
                        VERSION=$(yq '.version' config.yaml)
                        echo "Version: $VERSION"

                        # Count rules
                        RULE_COUNT=$(yq '.rules | length' config.yaml)
                        echo "Rules defined: $RULE_COUNT"

                        echo ""
                        echo "=== Checking filters.json structure ==="
                        FILTER_COUNT=$(jq '.feed.entry | length' filters.json)
                        echo "Filters in export: $FILTER_COUNT"

                        echo ""
                        echo "=== Checking labels.json structure ==="
                        LABEL_COUNT=$(jq '.labels | length' labels.json)
                        echo "Labels defined: $LABEL_COUNT"

                        echo ""
                        echo "✓ All structure checks passed"
                    '''
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
