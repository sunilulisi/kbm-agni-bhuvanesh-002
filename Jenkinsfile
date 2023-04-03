version="1.0.0"
//Make Backstage Scaffolder substitute using mustache template before publishing???
repository="docker-registry.mtk8s.io/voyagers/kbm-agni-bhuvanesh-002"
RepositoryName="kbm-agni-bhuvanesh-002"
privateregistry="docker-registry.mtk8s.io"
tag="latest"
image="${repository}:${version}.${env.BUILD_NUMBER}"
namespace="acnbackstage"
ENTITY_NAMESPACE = "default"
ENTITY_KIND = "Component"
ENTITY_NAME = "kbm-agni-bhuvanesh-002"
TECHDOCS_S3_TENANT_NAME = "http://minio.minio"
TECHDOCS_S3_BUCKET_NAME = "kbm-backstage-techdocs"
lbipaddr="44.205.217.18"

podTemplate(label: 'demo-customer-pod', cloud: 'kubernetes', serviceAccount: 'jenkins', imagePullSecrets: ['jenmtk8sdkrreg'], 
  containers: [
    containerTemplate(name: 'node16py311', image: 'nikolaik/python-nodejs:python3.11-nodejs16-bullseye', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'buildkit', image: 'docker-registry.mtk8s.io/moby/buildkit:latest', ttyEnabled: true, privileged: true, alwaysPullImage: true),
    containerTemplate(name: 'kubectl', image: 'roffe/kubectl', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'gitargocd', image: 'docker-registry.mtk8s.io/jenkins/inbound-agent:4.11.2-4-alpine-jdk11', ttyEnabled: true, alwaysPullImage: true, command: 'cat')
  ],
  volumes: [
    secretVolume(secretName: 'jenkins-docker-creds', mountPath: '/root/.docker'),
    configMapVolume(configMapName: 'kube-root-ca.crt', mountPath: '/var/tmp'),
    configMapVolume(configMapName: 'kube-intermediate-ca.crt', mountPath: '/var/tmp')
  ]) {

    node('demo-customer-pod') {
        stage('Prepare') {
            //
            // Since the GIT SCM is already performed as part of the Job Definition, Skipping Explicit Checkout
            // Needs to be tested
            //
            // git changelog: false, credentialsId: 'github-app-jenkins', poll: false, branch: 'main', url: 'https://github.com/kbmdev/backstage-acn-repo.git'
            checkout scm
            echo "SCM Checkout by using GitHub App"
        }
        
        stage('Build Tech Docs') {
            withCredentials([usernamePassword(credentialsId: 'minio-username-credentials', passwordVariable: 'pass', usernameVariable: 'user')]) {
                container('node16py311') {
                    sh """
                      # Minio related env variables
                       export AWS_ACCESS_KEY_ID="${user}"
                       export AWS_SECRET_ACCESS_KEY="${pass}"
                       export MINIO_ACCESS_KEY="${user}"
                       export MINIO_SECRET_KEY="${pass}"
                       export AWS_REGION="us-east-1"
                      
                      npm install -g @techdocs/cli
                      python -m pip install mkdocs-techdocs-core==1.*
                      techdocs-cli -V
                      techdocs-cli generate --no-docker --verbose
                      echo "successful generation of techdocs"
                      ls -latrh -R
                      techdocs-cli publish --publisher-type awsS3 --awsEndpoint $TECHDOCS_S3_TENANT_NAME --storage-name $TECHDOCS_S3_BUCKET_NAME --entity $ENTITY_NAMESPACE/$ENTITY_KIND/$ENTITY_NAME --awsS3ForcePathStyle true
                      echo "Build and Generate TechDocs Successful"
                    """
                    milestone(2)
                }
            }
        }
	    
	    stage('Sonarqube') {
	        container('gitargocd') {
            withSonarQubeEnv()
			{
		def scannerHome = tool 'SonarQubeScanner'
                    sh """
		    echo ${scannerHome}
		    ${scannerHome}/bin/sonar-scanner
			"""
		}
                   timeout(time: 10, unit: 'MINUTES') {
                   waitForQualityGate abortPipeline: true
	       }
            } 
                milestone(3)			
          }
	    
        stage('Node App Build and Package') {
            container('node16py311') {
                sh """
                  echo "Nodejs App Install and Build"
                  ls -latrh -R
                  npm install
                  npm run build && npm run bundle
                  mkdir -p dist
                  cp -R node_modules dist/
                  cp -R lib dist/
                  cp -R public dist/
                  cp -R package*.json dist/
                  cp -R src dist/
                  cp -R views dist/
                  ls -latrh dist
                """
                milestone(4)
            }
        }
	    
	    
        stage('Build Docker Image') {
            container('buildkit') {
                //Automatically the Docker Image Pull Secrets are Injected as part of POD Template
                sh """
                  ls -latrh
                  cat Dockerfile

                  #cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt .
                  #cp /usr/local/share/ca-certificates/kx-root-ca.crt .                  
                  #cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt ../../
                  #cp /usr/local/share/ca-certificates/kx-root-ca.crt ../../

                  echo -n "${lbipaddr} ${privateregistry}" >> /etc/hosts
                  hostname
                  cat /etc/hosts
                  cat /etc/resolv.conf
                  whoami
                  pwd
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=. --local dockerfile=. --output type=image,name=${image},push=true
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=. --local dockerfile=. --output type=image,name=${repository}:${tag},push=true
                """
                milestone(5)
            }
        }

    stage('Udate Appcode catalog yaml ') {
		withCredentials([usernameColonPassword(credentialsId: 'jenkins-cli-user-apitoken', variable: 'JENKINS_CLI_AUTH')]) {
            withCredentials([string(credentialsId: 'github-api-access-token', variable: 'GITHUB_API_TOKEN')]) {
            withCredentials([usernamePassword(credentialsId: 'github-app-jenkins', passwordVariable: 'ghpass', usernameVariable: 'ghuser')]) {
            withCredentials([usernamePassword(credentialsId: 'argocd-bs-bot-creds', passwordVariable: 'argoPass', usernameVariable: 'argoUser')]) {
            container('gitargocd')
			{
				sh """
                #! /bin/bash
				echo "Git Clone IAC Code Repo Project"
				
                git clone "https://github.com/kbmdev/${RepositoryName}-iac"
                cd "${RepositoryName}-iac"
                ls -latrh
                git config --global user.email "kbmdev@outlook.com"
                git config --global user.name "kbmdev"
				cat values.yaml
                # repository: docker-registry.mtk8s.io/voyagers/kbm-react-argocd-005
                # tag: 0.1 or anything
                # Replace !!!JUST!!! the tag value with the version ${version}.${env.BUILD_NUMBER}
                sed -i -e '/tag: /s/"[^"]*"/"${version}.${env.BUILD_NUMBER}"/g' values.yaml
                git status

                git add .
                git commit -m "IAC Code Values YAML Changes with Latest Image Version updates"
                git log
                git remote set-url origin https://kbmdev:${GITHUB_API_TOKEN}@github.com/kbmdev/${RepositoryName}-iac
                git remote -v
                git push origin master
                echo "Successfully Completed IAC Code Values YAML updates "	
               """
               milestone(6)
		}
		}
		}
		}
		}
	 
    }

        stage('Trigger ArgoCD Sync') {
            withCredentials([usernamePassword(credentialsId: 'argocd-bs-bot-creds', passwordVariable: 'argoPass', usernameVariable: 'argoUser')]) {
                container('gitargocd') {
                    echo "Perform Argo CD Sync"
                    sh """
                        # This trick is to pass Y to this pestering prompt even after passing the --insecure flag
                        # WARNING: server is not configured with TLS. Proceed (y/n)?
                        echo "y" | argocd login argocd-server.argocd --username=${argoUser} --password=${argoPass} --insecure --grpc-web
                        argocd version
                        argocd app list
                        argocd account get-user-info
                        argocd app sync ${RepositoryName}
                        argocd app list
                    """
                    milestone(7)
                }
            }
        }
    }
}

properties([[
    $class: 'BuildDiscarderProperty',
    strategy: [
        $class: 'LogRotator',
        artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10']
    ]
]);
