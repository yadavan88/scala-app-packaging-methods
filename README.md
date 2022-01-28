# Different Ways To package A Simple Scala App

## 1. Introduction
In this small article, let's look at different ways in which we can package our simple Scala application. 

## 2. Assumption
For this to work, I am assuming the below conditions are met:
- JDK is installed
- Scala version above 2.12 is installed. (For this specific example, I am using Scala 3.1.0)
- Probably an IDE like VS Code(with Metals) or IntelliJ IDEA or any other editor is installed

## 3. SBT Assembly
SBT is the most commonly used build tool for scala applications. And an executable jar file is the most simplest packaging we can create to run on multiple platforms. The only requirement is to have a jre. 

sbt-assembly is a very simple plugin that can be used to create jar files for your scala application. Let's look at it with an example. 

First of all, we need to create a simple sbt project. 

In the `plugins.sbt`, add the sbt-assembly dependency as:

```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "1.1.0")
```

After the re-importing the sbt build, let's add the required configurations in `build.sbt`.

```scala
assembly / assemblyJarName := "assemblyApp.jar"

```

Let's add a dependency for the project. This is just to show that the dependencies are also packaged within the jar and we can run the assembly jar without any additional configurations for classpath. In this particular example, I am using `os-lib` librarby as a dependency. 

Now, we can create our sample scala class:

```scala
package com.yadavan88.app

@main def mainMethod() =
  val osName = System.getProperty("os.name")
  val path = os.pwd.toString
  println(s"""
    | Hello from the packaged app! 
    | Current Path: ${path}
    """.stripMargin)

```

That's it. We are now ready to create a packaged jar with just a few configuraitons in `build.sbt`.

To create the package, we need to run the sbt command:
`sbt assembly`

This will create the jar file under the path `target/scala-3.1.0/app-packaging-assembly-1.0.2.jar` from the projecrt root. Since I am using scala-3.1.0, this is created under the above folder structure. For other scala versions, the path will change accordingly.

We can now execute the jar file from the directory as:
`java -jar app-packaging-assembly-1.0.2.jar`

By default sbt-assembly uses the project name and the version number to generate the jar file name.

Sbt-assembly also provides advanced configuration options. We can set a name for the jar file using:
```scala
assembly / assemblyJarName := "assemblyApp.jar"
```

Now, the jar file will be named as `assemblyApp.jar`.

Similarly, a lot more configuration options are available. More details are given in the [sbt-assembly github](https://github.com/sbt/sbt-assembly) page.

## 4. SBT Native Packager
SBT Assembly is a good plugin to create a jar file. But if there project is getting bigger, sbt-assembly might be a bit of difficult to manage. Especially, we might need to provide a lot of rules to handle deduplication. Also, it is not possible to create any other packaging formats using sbt-assembly. Here comes the use of `[sbt-native-packager](https://github.com/sbt/sbt-native-packager)`. 

SBT Native Packager allows us to create a wide variety of native packging formats like _exe_, _zip_, _msi_, _docker_, etc. 

Let's first add the plugin dependency to the `plugins.sbt` file:

```scala
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.7")
```

After importing the build, now we can add the relevant configurations in build.sbt:

```scala
enablePlugins(JavaAppPackaging) 
```

Now we can run the sbt command:

```scala
sbt universal:packageBin
```

This will create a zip package under `<project_root>/target/universal` which can then be copied to anywhere and unzipped. It will contain two scripts (windows and unix based scripts) under the directory `bin`. We can just execute this script to run our application. 

Instead of universal, we can also execute platform specific commands to generate the package. For example, to generate a debian package, we can run:
```scala
debian:packageBin
```

For windows:
```scala
sbt windows:packageBin
```

Similarly, we can create rpn package, mac package, graalvm native image etc. 

**However, please not that there may be pre-requisites to generate these platform specific packages. 
For example, to generate `msi` package for windows, the the system should have WIX toolkit installed. Similarly, for debian packaging, there should be relevant dpkg tools already installed**

We can add more configurations in the build.sbt to customise the package.

```scala
maintainer := "Yadukrishnan <yadavan88@gmail.com>"
Compile / mainClass := Some("com.yadavan88.app.mainMethod")
```

More such configuration options are available in the sbt-native-packager [documentation](https://sbt-native-packager.readthedocs.io/en/stable/formats/universal.html).


We can also use jlink based packaging in sbt-native-packager. jlink is a java tool which can identify and embed a minimal jre to the application. That means, the target system doesn't need to jave JRE/Java installed. To enable it, we can add this line to build.sbt:

```scala
enablePlugins(JlinkPlugin)

jlinkIgnoreMissingDependency := JlinkIgnore.only(
  "scala.quoted" -> "scala",
  "scala.quoted.runtime" -> "scala"
)

```

While building, there is a chance that you might be getting some errors due to unresolved dependencies. This can be manually suppressed by adding the ignore configurations for those which are not really needed for the runtime. In the previous example, the jlink was not able to find the library for `scala.quoted` package. Since we don't need it at runtime, we can ignore it. 
This might become tricky if the project gets bigger or if it uses some specific libraries which can't be packaged.

The jlink plugin now will copy all the dependencies and the jre libraries to a specific path, then packages the app using the universal plugin. So, we can now execute the command:
```scala
sbt universal:packageBin
```

The generated zip file will have the jre libraries within. This can be now copied and run in another system even without JRE installed. 

**NOTE: However, currently the generating system and the target system should be same. That means, if the app needs to be run in a windows machine, the jlink packaging also should be done in another windows machine.**

## 5. SBT Proguard
Proguard is a tool to optimise, obfuscate and package a java app. This is very useful in creating a shrinked application. [sbt-proguard](https://github.com/sbt/sbt-proguard) is a SBT plugin that can be used to package the scala application using proguard. 

First, we need to add the plugin to the plugins.sbt file:
```scala
addSbtPlugin("com.github.sbt" % "sbt-proguard" % "0.5.0")
```
Now we need to enable it in `build.sbt` and add the relevent configurations:
```scala
enablePlugins(SbtProguard)
Proguard / proguardOptions ++= Seq("-dontoptimize","-dontnote", "-dontwarn", "-ignorewarnings")
Proguard / proguardOptions += ProguardOptions.keepMain("com.yadavan88.app.mainMethod")
Proguard / proguardInputs := (Compile / dependencyClasspath).value.files
Proguard / proguardFilteredInputs ++= ProguardOptions.noFilter((Compile / packageBin).value)
```

__Please note the proguard options. These options are used by the proguard to optimise and obfuscate the code. In this case, the flag `dontoptimize` is used since proguard was corrupting the scala code while rewriting. More information regarding the proguard options are available [here](https://www.guardsquare.com/manual/configuration/usage)__


Now, let's run the sbt command:
```scala
sbt proguard
```
This will generate the executable jar file under tha path `<project_root>/target/scala-3.1.0/proguard`. We can now run the app like any other jar file, using the `java -jar <jarname.jar>`.

Please note the jar file size generated by the proguard plugin. In my case, it is just 1MB, whereas the jar file generated by assembly plugin is 7MB.

## 6. Scala-cli
[Scala-cli](https://scala-cli.virtuslab.org/) is a new command line tool which can be used to write and run scala programs. It can be used as a replacement for scala repl and ammonite repl/script. However, we can also use scala-cli to package small applications and make them executables. The advantage with scala-cli is that, it doesn't need sbt or any other plugins to create the packaging. 

First, let's install the scala-cli. The installation instructions are available [here](https://scala-cli.virtuslab.org/). After installed, let's verify it be running the command `scala-cli`. 

Now, let's create the class:
```scala
using scala "3.1.0"
package com.yadavan88.scalacli
import $dep.`com.lihaoyi::os-lib:0.7.8`
import os._

object ScalaCliApp {
  @main def app() = {
    val osName = System.getProperty("os.name")
    val path = os.pwd.toString
    println(s"""
                | Hello from the scala-cli packaged app!
                | Current Path: ${path}
                """.stripMargin)
  }
}

```
__This sample code is placed outside the src directory to avoid sbt compile issue since the scala-cli syntax is not compatible with sbt. The file is available under the path `<project_root>/scalacli_app`__

The line starting with `using` has a special meaning in scala-cli. It is called as `directives`, which are like configurations. In this example, it tells the scala-cli to use scala version `3.1.0` to compile and build the application. 
For additional dependecies, scala-cli uses the ammonite style ivy syntax using the keyword `$dep`. 

Note that, we can use any supported scala versions. Scala-cli internally uses [coursier](https://github.com/yadavan88/coursier-cheatsheets) to manage the dependencies.

Now, let's package our small app using the `package` task of scala-cli:
```scala
scala-cli package ScalaCliApp.scala -o cliapp --assembly
```

`--assembly` flag informs scala-cli to package all the dependencies along with our code. We can specify the app name using `-o` flag. The above command when executed will generate the app `cliapp`. We can execute the app using `./cliapp`.

**Note: scala-cli might not be a good option if there are many files and dependencies are involved.**

## 7. Conclusion
We have seen different ways in which we can package our scala application as executable packages. If you face any issues, feel free to create an issue here in github. I will try to help/solve the issue if I can. 

