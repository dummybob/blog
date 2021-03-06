---
title: Adding versions to your GitHub Actions
description: GitHub Actions are a powerful new feature for GitHub users, but they lack native versioning capabilities. In this blog post, we’ll see how to implement versioning.
author: matthew.casperson@octopus.com
published: 2019-08-29
visibility: public
metaImage: blogimg_versioning_githubactions.png
bannerImage: blogimg_versioning_githubactions.png
tags:
 - DevOps
---

![Illustration showing GitHub CI processes with versions](blogimg_versioning_githubactions.png)

[GitHub Actions](https://github.com/features/actions) are slowly rolling out to users as a beta. This new feature gives GitHub users a way to execute builds and deployments directly from their code using infrastructure managed by GitHub. This provides a lot of opportunities for developers, but while Actions are incredibly powerful and flexible, I immediately ran into the issue of versioning my builds.

Typically, a CI system will include some kind of incrementing counter that can be used as the patch release for any build. This means each build is automatically assigned a new number, and any resulting artifacts inherit a meaningful version. Unfortunately, in the current beta of GitHub Actions, there is no equivalent to a build number. The build environment exposes [information like Git SHAs, repositories, users, and events](https://developer.github.com/actions/creating-github-actions/accessing-the-runtime-environment/) but no build number.

This is inconvenient, to say the least, but there is a solution.

## Implementing GitVersion

[GitVersion](https://gitversion.readthedocs.io/en/latest/) is a tool that generates SemVer version numbers based on the tags in a Git repository. GitVersions is ideal for use with GitHub Actions because the Git repository itself is the source of truth for versioning, and no special tools beyond the Git client are required to manage the version numbers.

GitVersion also supplies a [Docker image](https://hub.docker.com/r/gittools/gitversion/), which can be directly used by GitHub Actions.

This means that, in theory, we have everything we need to generate meaningful version numbers as part of the GitHub Actions Workflow. In practice though, we still have some hoops to jump through.

## GitHub Actions and shared variables

GitHub Actions is based on the idea of individual jobs. These jobs can run on the underlying VM, or in a Docker container. Composing builds in this way offers great flexibility, but on the downside, it’s difficult to capture the output of one job and use it in another. CI servers often work around this problem by capturing the output with special markers. For example, TeamCity can watch for output in the format `##teamcity[setParameter name='env.whatever' value='myvalue']` and create a variable in response. Unfortunately, the beta of GitHub Actions does not provide any support for passing variables.

What we do have shared between jobs is the filesystem. Jobs that run directly on the VM can access the path `/home/runner/work/RepoName/RepoName/` (where `RepoName` is replaced with the name of the GitHub repository). Docker jobs have that same path mounted to `/github/workspace`.

We can use this shared file system to save the version generated by GitVersion, and then consume it in later steps.

## Example build

To see how this works in practice, take a look at the Workflow definition below. This is an example from a project that builds a Python AWS Lambda application:

```yaml
name: Python package

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Get Git Version
      uses: docker://gittools/gitversion:5.0.2-beta1-27-linux-centos-7-netcoreapp2.2
      with:
        args: /github/workspace /nofetch /exec /bin/sh /execargs "-c \"echo $GitVersion_FullSemVer > /github/workspace/version.txt\""
    - name: Set up Python 3.7
      uses: actions/setup-python@v1
      with:
        python-version: 3.7
    - name: Package dependencies
      run: |
        python -m pip install --upgrade pip
        cd hello_world
        pip download -r requirements.txt
        unzip \*.whl
        rm *.whl
        ls -la
    - name: Extract Octopus Tools
      run: |
        mkdir /opt/octo
        cd /opt/octo
        wget -O /opt/octo/octopus.zip https://download.octopusdeploy.com/octopus-tools/6.12.0/OctopusTools.6.12.0.portable.zip
        unzip /opt/octo/octopus.zip
        chmod +x /opt/octo/Octo
    - name: Pack Application
      run: >-
        /opt/octo/Octo pack .
        --outFolder /home/runner/work/AWSSamExample/AWSSamExample
        --basePath /home/runner/work/AWSSamExample/AWSSamExample/hello_world
        --id AwsSamLambda
        --version $(cat /home/runner/work/AWSSamExample/AWSSamExample/version.txt)
        --format zip
    - name: Push to Octopus
      run: >-
        /opt/octo/Octo push
        --server ${{ secrets.MATTC_URL }}
        --apiKey ${{ secrets.MATTC_API_KEY }}
        --package /home/runner/work/AWSSamExample/AWSSamExample/AwsSamLambda.$(cat /home/runner/work/AWSSamExample/AWSSamExample/version.txt).zip
        --overwrite-mode IgnoreIfExists

```

The first interesting part of this workflow is where we call the GitVersion docker image.

The trick here is to call GitVersion in a way that will save the resulting SemVer version to a file instead of printing it to the console output. We do this by setting the `/exec` argument to `/bin/sh`, and setting the `/execargs` argument to `"-c \"echo $GitVersion_FullSemVer > /github/workspace/version.txt\""`. These options result in GitVersion executing the shell, which writes the value of the environment variable `GitVersion_FullSemVer` (defined for us by GitVersion) to the file `/github/workspace/version.txt`.

The end result of this job is a file called `/github/workspace/version.txt` or `/home/runner/work/RepoName/RepoName/version.txt`, depending on whether or not the job is running inside a Docker container.

The environment variable `$GitVersion_FullSemVer` is just one of many provided by GitVersion. Any of the fields in the JSON below can be prefixed with `GitVersion_` and read as an environment variable.

```json
{                                                           
  "Major":0,
  "Minor":1,
  "Patch":0,
  "PreReleaseTag":"",
  "PreReleaseTagWithDash":"",
  "PreReleaseLabel":"",
  "PreReleaseNumber":"",
  "WeightedPreReleaseNumber":"",
  "BuildMetaData":55,
  "BuildMetaDataPadded":"0055",
  "FullBuildMetaData":"55.Branch.master.Sha.3903750b2aa5d84fd6004b2244cdb491f45520d9",
  "MajorMinorPatch":"0.1.0",
  "SemVer":"0.1.0",
  "LegacySemVer":"0.1.0",
  "LegacySemVerPadded":"0.1.0",
  "AssemblySemVer":"0.1.0.0",
  "AssemblySemFileVer":"0.1.0.0",
  "FullSemVer":"0.1.0+55",
  "InformationalVersion":"0.1.0+55.Branch.master.Sha.3903750b2aa5d84fd6004b2244cdb491f45520d9",
  "BranchName":"master",
  "Sha":"3903750b2aa5d84fd6004b2244cdb491f45520d9",
  "ShortSha":3903750,
  "NuGetVersionV2":"0.1.0",
  "NuGetVersion":"0.1.0",
  "NuGetPreReleaseTagV2":"",
  "NuGetPreReleaseTag":"",
  "VersionSourceSha":"0f692a38449b853d8a04aa891ac48e63ebec1add",
  "CommitsSinceVersionSource":55,
  "CommitsSinceVersionSourcePadded":"0055",
  "CommitDate":"2019-08-21"
}
```

```yaml
- name: Get Git Version
  uses: docker://mcasperson/gitversion:5.0.2-linux-centos-7-netcoreapp2.2
  with:
    args: /github/workspace /nofetch /exec /bin/sh /execargs "-c \"echo $GitVersion_FullSemVer > /github/workspace/version.txt\""
```

To consume the version, we will use the [Octopus CLI tools](https://octopus.com/docs/octopus-rest-api/octo.exe-command-line). In the following job, we download and extract the CLI package so it can be used in subsequent steps.

The Octopus CLI tools are also available as a [Docker image](https://hub.docker.com/r/octopusdeploy/octo), and so we could use these tools from the workflow with `uses: docker://octopusdeploy/octo:6.12.0`. However, calling docker images directly makes it difficult to use shell expansions, which we will need to extract the content of the file `version.txt` and pass it as a command-line argument. This is why we extract the tool locally instead:

```yaml
- name: Extract Octopus Tools
  run: |
    mkdir /opt/octo
    cd /opt/octo
    wget -O /opt/octo/octopus.zip https://download.octopusdeploy.com/octopus-tools/6.12.0/OctopusTools.6.12.0.portable.zip
    unzip /opt/octo/octopus.zip
    chmod +x /opt/octo/Octo
```

After the Octopus CLI tools are extracted, we can call them to package the application. Note how we use shell expansion with the argument `--version $(cat /home/runner/work/AWSSamExample/AWSSamExample/version.txt)` to read the value of the `version.txt` file created by GitVersion (`AWSSamExample` is the name of my GitHub repo). This is how we pass variables between jobs:

```yaml
- name: Pack Application
  run: >-
    /opt/octo/Octo pack .
    --outFolder /home/runner/work/AWSSamExample/AWSSamExample
    --basePath /home/runner/work/AWSSamExample/AWSSamExample/hello_world
    --id AwsSamLambda
    --version $(cat /home/runner/work/AWSSamExample/AWSSamExample/version.txt)
    --format zip
```

We use the Octopus CLI in a similar way to push the resulting application to the Octopus Server:

```yaml
- name: Push to Octopus
  run: >-
    /opt/octo/Octo push
    --server ${{ secrets.MATTC_URL }}
    --apiKey ${{ secrets.MATTC_API_KEY }}
    --package /home/runner/work/AWSSamExample/AWSSamExample/AwsSamLambda.$(cat /home/runner/work/AWSSamExample/AWSSamExample/version.txt).zip
    --overwrite-mode IgnoreIfExists
```

## Conclusion

GitHub Actions are a powerful new feature for developers, but there are some rough edges in the beta that you’ll have to work around for production scenarios. GitVersion provides a neat solution to the lack of build numbers and allows GitHub actions to interact with platforms like Octopus.
