title: How To Use Jenkins Parametered Build
date: 2017-02-13 22:31:40
tags: Jenkins
---

When we build staging and production environment in Jenkins, we might find out a lot of parts are similar and we just need the variable to switch build. That's the reason why we need this Jenkins parameterized build plugin. We can use it to define and switch some variables in your build script in order to increase flexibility.

## Preparation ##

Install the plugin                                                                                                                                                                [Parameterized Trigger plugin](https://wiki.jenkins-ci.org/display/JENKINS/Parameterized+Trigger+Plugin)

## Use ##

In your project, you need to select `This project is parameterized`
                                                                                                                                                                                  ![setting](/img/2017-02/parametered_build.png)

As we can see, Jenkins provides several ways to configure your parameter.
for example, we could use `Choice Parameter`

![build](/img/2017-02/build.png)

Then we could see sidebar has addition option called `Build with parameters`.
Click it and you could see there is a choice for setting the stage.

![building](/img/2017-02/building.png)

This is a useful skill to manage your project, try to use this plugin to reduce duplicate configurations!

Ref: [http://ithelp.ithome.com.tw/articles/10187358](http://ithelp.ithome.com.tw/articles/10187358)
