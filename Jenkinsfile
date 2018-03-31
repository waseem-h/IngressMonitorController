#!/usr/bin/groovy
@Library('github.com/stakater/fabric8-pipeline-library@add-git-tag')

def utils = new io.fabric8.Utils()

String chartPackageName = ""

toolsNode(toolsImage: 'stakater/pipeline-tools:1.5.1') {
    container(name: 'tools') {
        withCurrentRepo { def repoUrl, def repoName, def repoOwner, def repoBranch ->
            String srcDir = WORKSPACE + "/src"
            def kubernetesDir = WORKSPACE + "/kubernetes"

            def chartTemplatesDir = kubernetesDir + "/templates/chart"
            def chartDir = kubernetesDir + "/chart"
            def manifestsDir = kubernetesDir + "/manifests"
            
            def dockerImage = repoOwner.toLowerCase() + "/" + repoName.toLowerCase();
            
            def git = new io.stakater.vc.Git()
            def helm = new io.stakater.charts.Helm()
            def common = new io.stakater.Common()
            def chartManager = new io.stakater.charts.ChartManager()
            def docker = new io.stakater.containers.Docker()

            stage('Download Dependencies') {
                sh """
                    cd ${srcDir}
                    glide update
                    cp -r ./vendor/* /go/src/
                """
            }

            // if (utils.isCI()) {
            //     stage('CI: Test') {
            //         sh """
            //             cd ${srcDir}
            //             go test
            //         """
            //     }
            //     stage('CI: Publish Dev Image') {
            //         sh """
            //             cd ${srcDir}
            //             go build -o ../out/${repoName.toLowerCase()}
            //             cd ..
            //             docker build -t docker.io/${dockerImage}:dev .
            //             docker push docker.io/${dockerImage}:dev
            //         """
            //     }
            // } else if (utils.isCD()) {
                stage('CD: Build') {
                    sh """
                        cd ${srcDir}
                        go test
                        go build -o ../out/${repoName.toLowerCase()}
                    """
                }

                stage('CD: Tag and Push') {
                    print "Generating New Version"
                    def version = common.shOutput("jx-release-version --gh-owner=${repoOwner} --gh-repository=${repoName}")
                    sh """
                        echo "VERSION := ${version}" > Makefile
                    """

                    sh """
                        export VERSION=${version}
                        export DOCKER_IMAGE=${dockerImage}
                        gotplenv ${chartTemplatesDir}/Chart.yaml.tmpl > ${chartDir}/${repoName}/Chart.yaml
                        gotplenv ${chartTemplatesDir}/values.yaml.tmpl > ${chartDir}/${repoName}/values.yaml

                        helm template ${chartDir}/${repoName} -x templates/deployment.yaml > ${manifestsDir}/deployment.yaml
                        helm template ${chartDir}/${repoName} -x templates/configmap.yaml > ${manifestsDir}/configmap.yaml
                        helm template ${chartDir}/${repoName} -x templates/rbac.yaml > ${manifestsDir}/rbac.yaml
                    """
                    
                    git.commitChanges(WORKSPACE, "Bump Version to ${version}")

                    print "Pushing Tag ${version} to Git"
                    git.createTagAndPush(WORKSPACE, version)

                    print "Pushing Tag ${version} to DockerHub"
                    
                    docker.buildImageWithTag(dockerImage, "latest")
                    docker.tagImage(dockerImage, "latest", version)
                    docker.pushTag(dockerImage, version)
                    docker.pushTag(dockerImage, "latest")
                    // sh """
                    //     cd ${WORKSPACE}
                    //     docker build -t docker.io/${dockerImage}:${version} .
                    //     docker tag docker.io/${dockerImage}:${version} docker.io/${dockerImage}:latest
                    //     docker push docker.io/${dockerImage}:${version}
                    //     docker push docker.io/${dockerImage}:latest
                    // """
                }
                
                stage('Chart: Init Helm') {
                    helm.init(true)
                }

                stage('Chart: Prepare') {
                    helm.lint(chartDir, repoName)
                    chartPackageName = helm.package(chartDir, repoName)
                }

                stage('Chart: Upload') {
                    String cmUsername = common.getEnvValue('CHARTMUSEUM_USERNAME')
                    String cmPassword = common.getEnvValue('CHARTMUSEUM_PASSWORD')
                    chartManager.uploadToChartMuseum(chartDir, repoName, chartPackageName, cmUsername, cmPassword)
                }
            //}
        }
    }
}
