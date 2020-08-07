
# CICD for Jenkins use Pipeline Nexus and Ansible

部署流程：

- CI：nexus建立好存储包仓库，jenkinsfile调用nexusArtifactUploader模块上传jar包到nexus，jenkins可使用multibranch pipeline(本测试使用简单pipeline job方式)。
- CD： jenkins通过在project parameterized 中的Extensible Choice内调用nexus3 artifact choice parameter方式从nexus获取推送包，结合ansible进行部署。

### 1. 创建nexus仓库用于存储

#### 1.1. 创建repository

系统管理中选择repositories创建新仓库— repository-example

参数：

- type: hosted
- Version policy: Mixed
- Deployment policy: Allow redeploy

#### 1.2. 创建用户，授权管理仓库

直接用管理员上

### 2. 配置jenkins执行CI过程

#### 2.1. 安装插件

jenkins上传jar包我选择非maven仓库的方式上次，jenkins pipeline的[Nexus Artifact Uploader](https://wiki.jenkins-ci.org/display/JENKINS/Nexus+Artifact+Uploader)插件支持这种方式，官方示例：

```groovy
freeStyleJob('NexusArtifactUploaderJob') {
    steps {
        nexusArtifactUploader {
            nexusVersion('nexus2')
            protocol('http')
            nexusUrl('localhost:8080/nexus')
            groupId('sp.sd')
            version('2.4')
            repository('NexusArtifactUploader')
            credentialsId('44620c50-1589-4617-a677-7563985e46e1')
            artifact {
            artifactId('nexus-artifact-uploader')
            type('jar')
            classifier('debug')
            file('nexus-artifact-uploader.jar')
        }
        artifact {
            artifactId('nexus-artifact-uploader')
            type('hpi')
            classifier('debug')
            file('nexus-artifact-uploader.hpi')
            }
        }
    }
}
```

#### 2.2. jenkins创建credentials id

jenkins中需要创建credentials id，用户为nexus的用户，用于jenkinsfile连接nexus仓库。credentials id名称为nexus。

#### 2.3. 创建jenkins pipeline

新建jenkins pipeline job，编写jenkinsfile

```groovy
pipeline {
    agent {
        label "stateful"
    }
    environment {
        GIT_COMMIT_SHORT = ""
        // This can be nexus3 or nexus2
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "http"
        // Where your Nexus is running
        NEXUS_URL = "10.40.9.82:8081"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "repository-example"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus"
    }
    stages {
        stage('checkout code'){
            steps {
                checkout([$class: 'GitSCM', userRemoteConfigs: [[url: 'git@git.example.com:wayne/project.git', credentialsId: '06e172d1-43f6-4456-850d-56af9aeff5ed']], branches: [[name: 'master']]])
                script {
                    GIT_COMMIT_SHORT = sh (script: "git log -n 1 --pretty=format:'%h'", returnStdout: true)
                }
            }
        }
        stage("build") {
            steps {
                withDockerRegistry([credentialsId: 'jenkina-habor-credential', url: "https://wcr.example.com"]) {
                    withDockerContainer([image: 'wcr.example.com/wayne/maven:3.5.4-jdk-8']) {
                        sh "mvn package -DskipTests=true"
                    }
                }
            }
        }
        stage("publish to nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml";
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path;
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                            // Artifact generated such as .jar, .ear and .war files.
                            [artifactId: pom.artifactId,
                            classifier: "env.BRANCH-${GIT_COMMIT_SHORT}", //用这个字段标识版本号和分支
                            file: artifactPath,
                            type: pom.packaging],
                            // Lets upload the pom.xml file for additional information for Transitive dependencies
                            [artifactId: pom.artifactId,
                            classifier: "${GIT_COMMIT_SHORT}",
                            file: "pom.xml",
                            type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
    }
}
```

nexusArtifactUploader会从pom.xml中获取包的相关信息，然后上传到nexus

[详见插件文档](<https://wiki.jenkins.io/display/JENKINS/Nexus+Artifact+Uploader>)

### 3. 配置jenkins执行CD过程

#### 3.1. jenkins插件安装

获取nexus中的包需要使用parameterized中的Extensible Choice方法，需要安装nexus artifact插件供Choice Provider去选择配置

插件包：[Maven Artifact ChoiceListProvider (Nexus)](https://wiki.jenkins-ci.org/display/JENKINS/Maven+Artifact+ChoiceListProvider+Plugin)

配置jenkins获取仓库包

![jenkins artiface](/images/posts/jenkins_artiface.jpg)

点击左下角list进行测试，可以看到已存在于nexus仓库的包，在发布界面查看：

![部署](/images/posts/build_with_parameters.jpg)

后面执行ansible发布即可
