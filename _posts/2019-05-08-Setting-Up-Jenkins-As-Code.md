---
layout: post
---
Ok so our goal here is to deploy jenkins with the click of a button with our job configured and all. Our secret sauce for this will be jenkins **configuration-as-code-plugin**(JCasC) which allows you to define your jenkins setup in a YAML file or folder.
The problem is,  we want to use JCasC to configure jenkins but we need JCasC plugin installed ahead to be able to do that for us. Thankfully, we have a solution for that. We will be using Jenkins built-in process to install plugins.

#### Install plugins

```yaml
workflow-aggregator:latest
blueocean:latest
pipeline-maven:latest
configuration-as-code-support:latest
job-dsl:latest
```
For those installing jenkins using kubernetes, you will need to update your helm values [file](https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos/kubernetes-helm). Now, lets crank things up;
```yaml
#plugins.txt
workflow-aggregator:2.6
blueocean:1.16.0
pipeline-maven:3.6.11
configuration-as-code-support:1.14
job-dsl:1.74
workflow-job:2.32
credentials-binding:1.18
git:3.10.0
```

#### Build and Configure
```yaml
jenkins:
  systemMessage: "I did this using Jenkins Configuration as Code Plugin \n\n"
tool:
  git:
    installations:
    - home: "git"
      name: "Default"
  maven:
    installations:
    - name: "Maven 3"
      properties:
      - installSource:
          installers:
            - maven:
                id: "3.5.4"
jobs:
  - script: >
      pipelineJob('pipeline') {
          definition {
              cpsScm {
                  scriptPath 'Jenkinsfile'
                  scm {
                    git {
                        remote { url 'https://github.com/mkrzyzanowski/blog-001.git' }
                        branch '*/docker-for-mac'
                        extensions {}
                    }
                  }
              }
          }
      }
```
These are the plugins that we are trying to install as well as how we want our jenkins setup. Here is the Docker image build that takes care of installing  for us.
```yaml
#Dockerfile
FROM jenkins/jenkins:lts
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
```
Once the  image has been built, we need a way to let JCasC know the location of our configuration file( named jenkins.yaml in most cases).

+ Copy the **jenkins.yaml** file to /var/jenkins_home/. It looks for this file by default
+ Use **CASC_JENKINS_CONFIG** environmental variable to point to the file location and the location could be any of these;
    *  A  file path(/my/path/jenkins.yaml)
    *  A folder path(/my/path/jenkins_casc_configs/)
    *  A configuration file URL PATH(https://example.com/git/jenkins.yaml)

For this example, I will mount the jenkins.yaml to /var/jenkins_home with docker

```bash
$ docker run --name jenkins -p -d 8081:8080 -v $(pwd):/var/jenkins_home my_jenkins_image
Running from: /usr/share/jenkins/jenkins.war
webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
May 08, 2019 12:00:19 AM org.eclipse.jetty.util.log.Log initialized
INFO: Logging initialized @612ms to org.eclipse.jetty.util.log.JavaUtilLog
May 08, 2019 12:00:19 AM winstone.Logger logInternal
INFO: Beginning extraction from war file
May 08, 2019 12:00:40 AM org.eclipse.jetty.server.handler.ContextHandler setContextPath
WARNING: Empty contextPath
May 08, 2019 12:00:40 AM org.eclipse.jetty.server.Server doStart
INFO: jetty-9.4.z-SNAPSHOT; built: 2018-08-30T13:59:14.071Z; git: 27208684755d94a92186989f695db2d7b21ebc51; jvm 1.8.0_212-8u212-b01-1~deb9u1-b01
May 08, 2019 12:00:47 AM org.eclipse.jetty.webapp.StandardDescriptorProcessor visitServlet
INFO: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
May 08, 2019 12:00:47 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: DefaultSessionIdManager workerName=node0
May 08, 2019 12:00:47 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: No SessionScavenger set, using defaults
May 08, 2019 12:00:47 AM org.eclipse.jetty.server.session.HouseKeeper startScavenging
INFO: node0 Scavenging every 660000ms
Jenkins home directory: /var/jenkins_home found at: EnvVars.masterEnvVars.get("JENKINS_HOME")
May 08, 2019 12:00:50 AM org.eclipse.jetty.server.handler.ContextHandler doStart
INFO: Started w.@a50b09c{Jenkins v2.164.2,/,file:///var/jenkins_home/war/,AVAILABLE}{/var/jenkins_home/war}
May 08, 2019 12:00:50 AM org.eclipse.jetty.server.AbstractConnector doStart
INFO: Started ServerConnector@5a38588f{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
May 08, 2019 12:00:50 AM org.eclipse.jetty.server.Server doStart
INFO: Started @31513ms
May 08, 2019 12:00:50 AM winstone.Logger logInternal
INFO: Winstone Servlet Engine v4.0 running: controlPort=disabled
May 08, 2019 12:00:53 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started initialization
May 08, 2019 12:02:20 AM hudson.ClassicPluginStrategy createClassJarFromWebInfClasses
WARNING: Created /var/jenkins_home/plugins/job-dsl/WEB-INF/lib/classes.jar; update plugin to a version created with a newer harness
May 08, 2019 12:02:36 AM jenkins.InitReactorRunner$1 onAttained
INFO: Listed all plugins
May 08, 2019 12:02:58 AM jenkins.InitReactorRunner$1 onAttained
INFO: Prepared all plugins
May 08, 2019 12:02:58 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started all plugins
May 08, 2019 12:03:09 AM jenkins.InitReactorRunner$1 onAttained
INFO: Augmented all extensions
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.plugins.git.GitTool.name = Default
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.plugins.git.GitTool.home = git
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.tasks.Maven$MavenInstallation.name = Maven 3
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.tasks.Maven$MavenInstaller.id = 3.5.4
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.tools.InstallSourceProperty.installers = [{maven={}}]
May 08, 2019 12:03:10 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.tasks.Maven$MavenInstallation.properties = [{installSource={}}]
May 08, 2019 12:03:11 AM io.jenkins.plugins.casc.Attribute setValue
INFO: Setting hudson.model.Hudson@4fbfd7e4.systemMessage = I did this using Jenkins Configuration as Code Plugin 

Processing provided DSL script
May 08, 2019 12:03:15 AM javaposse.jobdsl.plugin.JenkinsJobManagement createOrUpdateConfig
INFO: createOrUpdateConfig for pipeline
May 08, 2019 12:03:16 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.plugins.git.GitTool.name = Default
May 08, 2019 12:03:16 AM io.jenkins.plugins.casc.impl.configurators.DataBoundConfigurator tryConstructor
INFO: Setting class hudson.plugins.git.GitTool.home = git
May 08, 2019 12:03:16 AM io.jenkins.plugins.casc.Attribute setValue
INFO: Setting hudson.plugins.git.GitTool$DescriptorImpl@7d18607f.installations = [GitTool[Default]]
....

```
Here is the screenshot of our newly configured Jenkins.

![Jenkins setup](https://thepracticaldev.s3.amazonaws.com/i/j2bb528d06hramjrlhrf.png)

Happy Automation!!!
