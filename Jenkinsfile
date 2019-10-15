#!groovy

def kubernetes_config = """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:12.10.0-alpine
    tty: true
    resources:
      limits:
        memory: "2Gi"
        cpu: "1"
      requests:
        memory: "2Gi"
        cpu: "1"
  - name: jnlp
    volumeMounts:
    - name: volume-known-hosts
      mountPath: /home/jenkins/.ssh
  volumes:
  - name: volume-known-hosts
    configMap:
      name: known-hosts
"""

def projectName = 'zos-resource-explorer'
def kubeLabel = projectName + '_pod_' + env.BRANCH_NAME + '_' + env.BUILD_NUMBER

pipeline {
    agent {
        kubernetes {
            label kubeLabel
            yaml kubernetes_config
        }
    }    
    options {
        timestamps()
        timeout(time: 3, unit: 'HOURS')
        skipDefaultCheckout(true)
        // skipDefaultCheckout(false)
        // disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '30'))
    }
    environment {
       branchName = "$env.BRANCH_NAME"
       workspace = "$env.WORKSPACE"
    }
    stages {
        stage('Install & Test') {
            environment {
                npm_config_cache = "$env.WORKSPACE"
            }
            steps {
                container('node') {
                    // sh "npm ci"
                    // sh "npm test"
                }
            }
        }
        stage('Package') {
            environment {
                npm_config_cache = "$env.WORKSPACE"
            }
            steps {
                container('node') {
                    // sh "npm run webpack-production"
                    // sh "npx vsce package"
                    // sh "mv *zosexplorer*.vsix zosexplorer_latest.vsix"
                    sh "wget https://ci.eclipse.org/che4z/job/LSP%20for%20COBOL/job/release-0.8.3/lastSuccessfulBuild/artifact/clients/cobol-lsp-vscode-extension/cobol-language-support_latest.vsix"
                }
            }
        }
        stage('Archive Artifacts') {
            steps {
                container('node') {
                    archiveArtifacts '*.vsix'
                }
            }
        }
        stage('Deploy') {
            environment {
                sshChe4z = "genie.che4z@projects-storage.eclipse.org"
                project = "download.eclipse.org/che4z/snapshots/$projectName"
                url = "$project/$branchName"
                deployPath = "/home/data/httpd/$url"
            }
            steps {
                script {
                    if (branchName == 'master' || branchName == 'development') {
                        container('jnlp') {
                            sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
                                sh '''
                                ssh $sshChe4z rm -rf $deployPath
                                ssh $sshChe4z mkdir -p $deployPath
                                scp -r $workspace/*.vsix $sshChe4z:$deployPath
                                '''
                                echo "Deployed to https://$url"
                            }
                        }
                    } else {
                        echo "Deployment skipped for branch: $branchName"
                    }
                }
            }
        }
    }
}
