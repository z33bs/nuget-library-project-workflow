# WorkFlow for new Nuget Library Project

Summary:

* Initialise and co-ordinate: vs, kraken, git, azure
* Continuous integration: build pipeline and release pipeline
* Publishing to MyGet, and Nuget

***

## Repos

Create Repo in GitHub

* Include Readme and Licence
* Don't include gitIgnore as VS will do this (we would overwrite anyway)

Clone repo into VS using Checkout

Create new Blank Solution in VS

* In the root folder of the repo
* Include .gitignore file

In solution, create new Project

* Put in /src/ProjectName
* Choose NetStandard for library. Shareable across all .NET

In VS, create solution items folder

* Add existing files License, Readme etc

Open Repo in GitKraken

* VS version control won't see its own .sln file but Kraken will
* Commit and push to GitHub remote master to ensure all works

Initialise GitFlow in GitKraken

* Switch to develop branch asap

Link to Azure Build Pipelines

* Using organisation guy-antoine (owned by zeebsmail@gmail)
* New Project
  * Settings>PipeLines>Service Connections: Add GitHub
  * For Authentication: Select Personal Access Token
    * Follow link that takes you to place in GitHub where create one. Create a new one, only have to give repo permissions. Copy the key and use to authenticate. Note this will disappear once you've copied the key. But wont need it again.
* Now go out of settings... and onto Pipelines Landing Page. Create a new Pipeline
  * Just choose a basic minimal one. We will replace the code with boilerplate that i've standardised below

***

## Build Pipeline

I like to run two pipelines:

* Alpha channel purely for my own consumption. This will be triggered by pushes to the develop branch
  * Package Version will simply be based on the date
* Beta & Release pipeline that will be triggered by pushes to the Master branch
  * Semiver convention for versioning (with autoincrement for build revision)

Tips:

* Use variable names to make yaml easily re-usable
* Use specific project and solution names instead of pattern matching *.csproj because pipeline will get confused when more than one project (e.g. test project included too)
* Use specific build configuration
* Remember to publish artifacts! Otherwise the build won't be accessible from your release workflow
* Remember there is also a copy file option
* Simpler is better (mostly). Use defaults where can. Dont add stuff for the sake of it. Eg if you're coding solo and do most of your testing locally, no need to slow the pipeline by running tests.

Preparing the csproj file

* You can use Options > Nuget to enter meta-data

  * Once you're done, go edit the csproj. You can remove unused stuff like PackageVersion, because we'll controll this in the pipeline
  * If VS automatically installs nuget-builder dependency, remove this. We don't need it and it may interfere with the pipeline.

* For nuget, you may want to add:

  * ```xml
    <PackageIcon>icon.png</PackageIcon>
    
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <!-- OR -->
    <!-- <PackageLicenseFile>LICENSE.md</PackageLicenseFile> -->
    
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <RepositoryUrl>https://github.com/z33bs/SmartDi</RepositoryUrl>
    
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    ```

  * because license url and icon url are depreciated

Sidenote on versioning: New csproj will autogenerate assemblyinfo code. Most important is assembly version, which is used by project reference to determine the correct version. This by convention is only changed if there are changes affecting backwards compatability. Assembly-file-version is only used by file system in windows. Information-version is unused, it was adopted by nuget for package version... but now there's an explicit package version.

### Alpha

Very basic flow:

* DotNet build
* DotNet pack
  * Output to artifactstagingdirectory
* Publish artifacts from staging to default location

```yaml
# Build pipeline for Develop Branch

# This sets $(Build.BuildNumber)
name: $(Date:yyyyMMdd).$(Rev:r)

# Set project names here
variables:
  projectName: SmartDi.csproj

trigger:
- develop

pool:
  vmImage: 'ubuntu-latest'

steps:

# Implicit Restore run before build
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    projects: '**/$(projectName)'
    arguments: '--configuration Release'

- task: DotNetCoreCLI@2
  inputs:
    command: 'pack'
    packagesToPack: '**/$(projectName)'
    configuration: 'Release'
    packDirectory: '$(Build.ArtifactStagingDirectory)/alpha'
    nobuild: true
    versioningScheme: 'byEnvVar'
    versionEnvVar: 'Build.BuildNumber'

# Note, we're outputing to */alpha to differentiate pre-release

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
    publishLocation: 'Container'

```



*This is old code using MSBuild task*

```yaml
# Build pipeline for Develop Branch

# This sets $(Build.BuildNumber)
name: $(Date:yyyyMMdd).$(Rev:r)

# Set project names here
variables:
  solutionName: SmartDi.sln
  projectName: SmartDi.csproj

trigger:
- develop

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: NuGetCommand@2
  inputs:
    command: 'restore'
    restoreSolution: '**/$(solutionName)'
    feedsToUse: 'select'

# Note, we're outputing to */alpha to differentiate pre-release
- task: MSBuild@1
  inputs:
    solution: '**/$(projectName)'
    configuration: 'Release'
    msbuildArguments: '-t:restore;build;pack -p:PackageOutputPath=$(Build.ArtifactStagingDirectory)/alpha -p:PackageVersion=$(Build.BuildNumber)'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
    publishLocation: 'Container'

```

### Beta & Release

Flow is:

* Build
* Pack with -pre suffix => put in /beta folder 
* Pack => put in /public folder
* Publish Artifacts
* Optional to add test run and coverage report
  * Not detailed here but maybe try use dotnet command, similar to my local workflow

```yaml
# Build pipeline for Master Branch

# This sets $(Build.BuildNumber)
name: 1.0.$(Rev:r)

# Set project names here
variables:
  projectName: SmartDi.csproj

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:

# Implicit Restore run before build
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    projects: '**/$(projectName)'
    arguments: '--configuration Release'

# Package with -pre version for /beta
- task: DotNetCoreCLI@2
  inputs:
    command: 'pack'
    packagesToPack: '**/$(projectName)'
    configuration: 'Release'
    packDirectory: '$(Build.ArtifactStagingDirectory)/beta'
    nobuild: true
    versioningScheme: 'off'
    buildProperties: 'PackageVersion=$(Build.BuildNumber)-pre'

# Package public version off same build
- task: DotNetCoreCLI@2
  inputs:
    command: 'pack'
    packagesToPack: '**/$(projectName)'
    configuration: 'Release'
    packDirectory: '$(Build.ArtifactStagingDirectory)/public'
    nobuild: true
    versioningScheme: 'off'
    buildProperties: 'PackageVersion=$(Build.BuildNumber)'


- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
    publishLocation: 'Container'

```

If you want to add a badge, goto the pipeline where you can browse runs: on menu, select Badge

![image-20200518135649207](https://cdn.jsdelivr.net/gh/z33bs/MyImageServer@master/2020/05/18_1356_KB9huN.png)

***

# Release Pipeline

### Alpha (MyGet)

Very lightweight for rapid use

![image-20200518131252561](https://cdn.jsdelivr.net/gh/z33bs/MyImageServer@master/2020/05/18_1312_D0qqvc.png)

* Artifacts - Trigger

  * use branch filter: develop

* Pre stage

  * Default trigger (after release)
  * Add Task: Nuget 
    * Modify action to Push (see example setup below)
  * For package path, browse the artifacts folder and select manually
    * then modify to generalise using /**/ for path, and *. for filename

* MyGet settings

  * Remember your's is self hosted in Azure Portal
  * Sign in, goto MyGet resource > click on Manage link
    * This will take you to MyGet website and log you in with azure credentials
    * Here, take note of the API key. If you go to the feed section, you will get both the feed url and the api key

![image-20200517133425375](https://cdn.jsdelivr.net/gh/z33bs/MyImageServer@master/2020/05/17_1334_wBw2lr.png)

### Public (Nuget)

Can get quite heavy if include automated testing etc

![image-20200518125948767](https://cdn.jsdelivr.net/gh/z33bs/MyImageServer@master/2020/05/18_1259_OwZuCh.png)

* Artifacts - Trigger
  * use branch filter: master
* Pre stage
  * Default trigger (after release)
  * No pre-deployment vetting (if I've merged to master this is the vetting)
  * Same setup as MyGet but use Nuget instead
  * Will need to regenerate API key in nuget as it only shows this once (if using one key need to update for all projects)
  * Note path to packages will be different
* Public stage
  * Trigger - After Stage
  * Pre-deployment condition: approval required
  * Task = same as previous stage, with different package path

You may want to add a badge on your github readme. To do this, select Options>Integrations

![image-20200518132729582](https://cdn.jsdelivr.net/gh/z33bs/MyImageServer@master/2020/05/18_1327_DHBhoh.png)

## More badge examples

```markdown
[![NuGet](https://buildstats.info/nuget/XamarinFormsMvvmAdaptor?includePreReleases=true)](https://www.nuget.org/packages/XamarinFormsMvvmAdaptor/)  
![Coverage](https://img.shields.io/azure-devops/coverage/guy-antoine/xamarin-forms-mvvm-adaptor/2?label=Coverage)  
[![Build Status](https://dev.azure.com/guy-antoine/xamarin-forms-mvvm-adaptor/_apis/build/status/z33bs.xamarin-forms-mvvm-adaptor%20(1)?branchName=master)](https://dev.azure.com/guy-antoine/xamarin-forms-mvvm-adaptor/_build/latest?definitionId=2&branchName=master) 
[![Dev. Feed](https://img.shields.io/badge/Dev%20Feed-MyGet-yellow)](https://www.myget.org/feed/zeebz-open-source/package/nuget/XamarinFormsMvvmAdaptor)
```

