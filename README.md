# squbs Training for Java Users

### Prerequisites

1. Have a laptop ready. Any operating system is good.
2. Update Java version. Tested versions are JDK 1.8.0_60 and later. Any later version or the latest on Oracle site.
3. Install IntelliJ Idea Community Edition, or upgrade to the latest version. It’s free. Make sure you have a version of late 2017 at least as some of the interfaces are much simplified. You can get it here: https://www.jetbrains.com/idea/download/
4. Start IntelliJ Idea and install the Scala plugin. This is needed for our build tool to work.
5. Install sbt from its [download site](https://www.scala-sbt.org/download.html).
6. Use [squbs-java-seed](https://github.com/paypal/squbs-java-seed.g8) to create a new project by running `sbt new paypal/squbs-java-seed.g8`
7. Use `git clone` to clone your project into your laptop/desktop. That’s better done from the terminal.
8. From IntelliJ, open the project by select `File`->`Open`, then select the `build.sbt` file at the root level of the cloned git directory (there are many `build.sbt` files in other sub-projects. Make sure to choose the one in the root directory of the cloned project).
9. IntelliJ will prompt whether you want to open as `File` or `Project`. Select `Project`.
10. Next is the `Import` screen.
   *  Choose to import library sources (a checkbox item) as well.
   *  Check Project JDK and make sure it is the version you wanted. Again, we need to use JDK 1.8.0_60 or later. To register a new JDK to IntelliJ, press the `New` button. Select JDK, then locate and select the JDK of your choice.
   *  Press `Continue`
11. At this stage, IntelliJ will download and resolve all libraries needed for your project. Depending on network bandwidth, this can take 2 minutes to 3 hours (I’ve seen it sometimes when training in Chennai and some computers seem to just have a very slow network plus the low bandwidth locally). Let this finish.
12. We should be ready to start the tutorial now.

### Getting Started

Start the tutorial from the [docs](docs) folder. It is organized in order.
