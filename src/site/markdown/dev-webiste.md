# Contributing to the Flex Website

This document describes how the Maven version of the Flex Website is created and why it is done that way.

## The Git repo and its structure
 
Currently the flex-site repo, which is available at:
    
    https://git-wip-us.apache.org/repos/asf/flex-site.git
    
Because we haven't quite decided which path to go in migrating away from the Apache CMS and SVN to a GIT-based version, we have decided to create branches for each approach and decide later on which of them we will be using. Currently the only approach being worked on is one based upon Apache Maven using the `maven-site-plugin`

Contains several branches:

- `master` (Is just there to prevent problems when checking out)
- `maven-site` (Contains this document and is a version of the Apache Flex website, which is built from markdown content using the maven-site-plugin in a Maven build)
- `asf-site` (Contains the HTML the maven-site project creates when running the site:deploy goal. This it the static code of the Apache Flex website which will be served by the Apache Webservers)

## How the site is generated

Currently the projects are able to produce content in their own repositories. The main Apache Foundation webservers server their content from a SVN repository hosted at:

    https://svn.apache.org/repos/infra/websites/production/{project-name}
    
The projects however cannot publish direct into this repository. Therefore the Infra team have created tools to sync content from projects repositories with their sub-branch of that SVN repo:

- For SVN based sites this is svnpubsub
- For GIT based sites this is gitpubsub
- The Apache CMS which sort of generates content from a SVN repo and automatically publishes that so some sort of staging area from which you can review changes and release them to production using a web-ui (This is the way the Apache Flex website is currently created)

As we are planning on using GIT and therefore we have to migrate away from the Apache CMS, which can only deal with SVN based repositories.

The `maven-site` branch contains a simple Maven project which utilizes the `maven-site-plugin` to generate the Website and render `Markdown` content as nice looking HTML Website.
This static content is then checked in to the `asf-site` branch of the same GIT repository.

As this checking in requires commit permissions, we cannot run this job on the normal ASF Jenkins instances. The only build system that allows automatic commits to git is `buildbot`. Setting up a job here is a little more tricky than with jenkins as you have to provide the job definition by adding a config file to a SNV repo hosted at:

    https://svn.apache.org/repos/infra/infrastructure/buildbot/aegis/buildmaster/master1/projects

Currently this config file for building the `maven-site` version of the Flex website looks like this:

````
# This is the config file for generating the website of the flex project.

#########################################################################################
# Add a new scheduler for the current job
#########################################################################################

# Check the "maven-site" branch in the "flex-site" git repo for changes and trigger
# the "maven-flex-site" job if there are changes
c['schedulers'].append(SingleBranchScheduler(name="on-maven-flex-site-commit",
                    change_filter=filter.ChangeFilter(branch='maven-site' , project='flex-site'),
                    treeStableTimer=2,
                    builderNames=["maven-flex-site"]))



#########################################################################################
# Define the Job
#########################################################################################

# Build Factory (Define which steps have to be performed to execute the build)
flexSiteJobFactory = factory.BuildFactory()
# 1. Checkout the "flex-site" branch of the flex-site repo
flexSiteJobFactory.addStep(Git(
    repourl="https://git-wip-us.apache.org/repos/asf/flex-site.git",
    branch='maven-site',
    workdir="build",
    retry=(10, 5), # retry 5 times with a 10 second delay
    retryFetch=True,
    mode='full',
    method='fresh'
    ))
# 2. Run a Maven "clean site-deploy" build on the checked out repo.
flexSiteJobFactory.addStep(Compile(command=["mvn" , "clean" , "site-deploy"]))



#########################################################################################
# Define the Builder
#########################################################################################

# Run the job with the name "maven-flex-site" on the agent with the name "bb_slave1_ubuntu"
# Use the working directory "maven-flex-site-build-dir" and report the outcome in the
# category "maven-flex-site-category"
flexSiteBuild = {'name': "maven-flex-site",
      'slavename': "bb_slave1_ubuntu",
      'builddir': "maven-flex-site-build-dir",
      'factory': flexSiteJobFactory,
      'category': "maven-flex-site-category",
     }



#########################################################################################
# Add the new flexSiteBuild to the other builders (
#########################################################################################

c['builders'].append(flexSiteBuild)



#########################################################################################
# Define how the status should be reported
#########################################################################################

c['status'].append(mail.MailNotifier(fromaddr="buildbot@apache.org",
                                      extraRecipients=["commits@flex.apache.org"],
                                      sendToInterestedUsers=False,
                                      mode="change",
                                      categories=["maven-flex-site-category"]))
````

In order to make `buildbot` find the new config, you need to add it to a file called `projects.conf` in the same directory as the config file. 
As soon as you commit, `buildbot` will automatically process the config file and report any problems via Email.

In above script I defined a build-job called `maven-flex-site` so the url to view it's state is:

    https://ci.apache.org/builders/maven-flex-site/

Unfortunately you can't directly click on a button to run the build, but you can always trigger a build manually using IRC.
In order to do this, you need to 
1. connect to the channel `#asftest` on a `freenode` IRC server (I used `chat.freenode.net`).
2. send the following message: `pony-bot: force build maven-flex-site because ponies`

(Don't ask me about the ponys. I was told there are multiple bots listening and `pony-bot` is one that is able to trigger the build ... and the `because ponies` makes Pono happy, because he seems to like ponies a lot ;-) )

At least after this you should see some results in the job overview page.

## Open Issues

1. Currently the `buildbot` job doesn't automatically start as soon as someone checks in something into the `maven-site` branch.
2. Currently the automatic commit used by the `maven-site-plugin` doesn't seem to allow committing, so I'll have to figure out that one.
3. As soon the build works and the content is migrated, we have to request Infa to setup `gitpubsub` to sync the content of the `asf-site` with the Flex projects content in the Infra webserver SVN repo.

