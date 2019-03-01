# squbs Training for Java Users

### Prerequisites

1. Have a laptop ready. Either Windows or Mac is good. You can also use a desktop if you attend from your desk.
2. Update Java version. Supported versions are JDK 1.8.0_60 and 1.8.0_121. Any later version or the latest on Oracle site.
3. Install IntelliJ Idea Community Edition, or upgrade to the latest version. It’s free. Make sure you have a version of late 2017 at least as some of the interfaces are much simplified. You can get it here: https://www.jetbrains.com/idea/download/
4. Start IntelliJ Idea and install the Scala plugin. This is needed for our build tool to work.
5. Go to Altus, create a new squbs “Service” project. At this time you’ll have chosen a github repo to create your project. You don’t need to have a CI or a User Stage for this lesson. Uncheck those.
6. Use `git clone` to clone your project into your laptop/desktop. That’s better done from the terminal.
7. From IntelliJ, open the project by select `File`->`Open`, then select the `build.sbt` file at the root level of the cloned git directory (there are many `build.sbt` files in other sub-projects. Make sure to choose the one in the root directory of the cloned project).
8. IntelliJ will prompt whether you want to open as `File` or `Project`. Select `Project`.
9. Next is the `Import` screen.
   *  Choose to import library sources (a checkbox item) as well.
   *  Check Project JDK and make sure it is the version you wanted. Again, we need to use JDK 1.8.0_60 or later. To register a new JDK to IntelliJ, press the `New` button. Select JDK, then locate and select the JDK of your choice.
   *  Press `Continue`
10. At this stage, IntelliJ will download and resolve all libraries needed for your project. Depending on network bandwidth, this can take 2 minutes to 3 hours (I’ve seen it sometimes when training in Chennai and some computers seem to just have a very slow network plus the low bandwidth locally). Let this finish.
11. We should be ready to start the tutorial now.

### Getting Started

Start the tutorial from the [docs](docs) folder. It is organized in order.