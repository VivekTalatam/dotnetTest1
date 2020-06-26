# WebApplication
Sample Application for CI/CD


## Tools:
* CI: Jenkins 
* Source Code: GitHub (Cloud)
* Issue Management: GitHub (Cloud)
* Tech stack: ASP.Net MVC, EF, JQuery, Bootstrap, Webpack
* Database: SQL Server
* Web Server: IIS
* Testing: MSTest, Specflow, Selenium, Moq
* Build Tools: MSBuild, Nuget, MSDeploy

## jenkinsfile:

### Environment section
Sets RELEASE_VERSION and tools paths.
```
environment{
    RELEASE_VERSION = "1.0.1"
    VSTest = tool 'vstest'	
    MSBuild = tool 'msbuild'
    Nuget = 'C:\\Program Files (x86)\\Jenkins\\nuget.exe'
    MSDeploy = "C:\\Program Files (x86)\\IIS\\Microsoft Web Deploy V3\\msdeploy.exe"
}
```
### Stage Get Source
Gets source code and notifies to Slack and Github.
```
stage('Get Source'){
  steps{
    slackSend (color: '#FFFF00', message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    setBuildStatus("PENDING: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", "PENDING");
    checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: 'https://github.com/alanmacgowan/WebApplication.git']]])
  }
}
```
<img src="https://github.com/alanmacgowan/alanmacgowan.github.io/blob/master/GITHUB%20NOTIF%20PENDING.png" />

### Stage Restore dependencies

### Nuget
Restores nuget packages, runs in parallel.
```
stage('Restore'){
    steps{
        bat "\"${Nuget}\" restore WebApplication.sln"
    }
}
```
### Stage NodeJS
Installs npm dependencies and runs webpack, runs in parallel.
```
stage('NodeJS'){
    steps{
        nodejs(nodeJSInstallationName: 'Node') {
            dir('WebApplication')
            {
                bat 'npm install && npm run build:prod'
            }
        }
    }
}	
```
### Stage Set Assembly Version
Changes Assemblyversion number with current Build number, using Powershell script, runs in parallel.
```
stage('Set Assembly Version') {
  steps {
        dir('WebApplication\\Properties')
        {
            setAssemblyVersion()
        }
   }
}
...
void setAssemblyVersion(){
	powershell ("""
	  \$PatternVersion = '\\[assembly: AssemblyVersion\\("(.*)"\\)\\]'
	  \$AssemblyFiles = Get-ChildItem . AssemblyInfo.cs -rec

	  Foreach (\$File in \$AssemblyFiles)
	  {
		(Get-Content \$File.PSPath) | ForEach-Object{
			If(\$_ -match \$PatternVersion){
				'[assembly: AssemblyVersion("{0}")]' -f "$RELEASE_VERSION.$BUILD_NUMBER"
			} Else {
				\$_
			}
		} | Set-Content \$file.PSPath
	  }
    """)
}

```
### Stage Build & Package
Build and packages WebApplication for latter deployment. The artifact is saved as part of the build in Jenkins.
```
stage('Build & Package') {
    steps {
					    bat "\"${MSBuild}\" WebApplication.sln /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:SkipInvalidConfigurations=true /t:build /p:Configuration=QA /p:Platform=\"Any CPU\" /p:DesktopBuildPackageLocation=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\""
    }
}
```
### Stage Unit test
Runs unit tests.
```
stage('Unit test') {
    steps {
        dir('WebApplication.Tests.Unit\\bin\\Release')
        {
            bat "\"${VSTest}\" \"WebApplication.Tests.Unit.dll\" /Logger:trx;LogFileName=Results_${env.BUILD_NUMBER}.trx /Framework:Framework45"
        }
        step([$class: 'MSTestPublisher', testResultsFile:"**/*.trx", failOnError: true, keepLongStdio: true])
    }
}
```
### Stage Deploy to QA
Deploys package to QA IIS site, web.config is reeplaced by config file in Jenkins.
```
stage('Deploy to QA') {
  steps {
      bat "\"${MSDeploy}\" -source:package=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\"  -verb:sync -dest:auto -allowUntrusted=true -setParam:name=\"IIS Web Application Name\",value=\"TestAppQA\""
      configFileProvider([configFile(fileId: 'web.QA.config', targetLocation: 'C:\\Jenkins_builds\\sites\\qa\\web.config')]) {}
  }
}
```
### Stage Smoke Test QA
Smoke Test QA IIS site, using Powershell script.
```
stage('Smoke Test QA') {
  steps {
    smokeTest("http://localhost:8091/")
  }
}
...
void smokeTest(String url){
	powershell ("""
		\$result = Invoke-WebRequest $url
		if (\$result.StatusCode -ne 200) {
			Write-Error \"Did not get 200 OK\"
		} else{
			Write-Host \"Successfully connect.\"
		}
	""")
}
```
### Stage Acceptance test
Runs acceptance testsusing local IIS site.
```
stage('Acceptance test') {
    steps {
        dir('WebApplication.Tests.Acceptance\\bin\\Release')
        {
            bat "\"${VSTest}\" \"WebApplication.Tests.Acceptance.dll\" /Logger:trx;LogFileName=Results_${env.BUILD_NUMBER}.trx /Framework:Framework45"
        }
        step([$class: 'MSTestPublisher', testResultsFile:"**/*.trx", failOnError: true, keepLongStdio: true])
    }
}
```
### Stage Deploy to Prod
Waits for input from user to deploy.
Deploys package to Prod IIS site, web.config is reeplaced by config file in Jenkins.
```
stage('Deploy to Prod') {
  input {
    message 'Deploy to Prod?'
    ok 'Yes'
  }
  steps {
      bat "\"${MSDeploy}\" -source:package=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\"  -verb:sync -dest:auto -allowUntrusted=true -setParam:name=\"IIS Web Application Name\",value=\"TestAppProd\""
      configFileProvider([configFile(fileId: 'web.Prod.config', targetLocation: 'C:\\Jenkins_builds\\sites\\prod\\web.config')]) {}
  }
} 
```
### Stage Smoke Test Prod
Smoke Test Prod IIS site, using Powershell script.
```
stage('Smoke Test Prod') {
  steps {
    smokeTest("http://localhost:8092/")
  }
}
```
### Post Section
Notification and archiving.
```
post { 
  failure { 
    slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    setBuildStatus("FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", "FAILURE");
  }
  success{
    slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    setBuildStatus("SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", "SUCCESS");
  }
  always {
    archiveArtifacts "artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip"
  }
}
```
## Includes:
* Jenkinsfile: for multibranch pipeline on Jenkins, triggers from SCM on PR merged to develop branch, builds and deploys to local IIS Server.
* web-deploy.ps1: powershell script that builds, deploys to file system and package web application.
* web-publish.ps1: powershell script that builds, packages and deploys to IIS web application.
* azure-pipelines.yml: configuration file for Azure Devops pipeline.
* Docker/Dockerfile: Dockerfile for linux image with jenkins installed and plugins.
* Docker/Dockerfile_Windows: Dockerfile for windows image(based on: https://blog.alexellis.io/continuous-integration-docker-windows-containers/)

