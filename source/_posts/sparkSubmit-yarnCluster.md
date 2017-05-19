---
title: sparkSubmit_yarnCluster
date: 2016-06-28 18:59:51
tags: [spark,源码]
---

&nbsp;&nbsp;&nbsp;&nbsp;spark提交任务的时候都是通过spark-submit提交的，而spark-submit提交的方式最终执行的都是sparkSubmit这个类,常见的有``yarn-cluster``、``rest``、``client``。这里主要看下``yarn-cluster``

```scala
  private[spark] def printVersionAndExit(): Unit = {
    printStream.println("""Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version %s
      /_/
                        """.format(SPARK_VERSION))
    printStream.println("Type --help for more information.")
    exitFn(0)
```
<!--more-->
打开这个代码，一看到这个图案你就能想得到，什么``spark-shell``、``spark-submit``、``kill``等其实都是通过这个类来实现的。

main方式如下：

```
val appArgs = new SparkSubmitArguments(args)
    if (appArgs.verbose) {
      // scalastyle:off println
      printStream.println(appArgs)
      // scalastyle:on println
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    }
```

main方法里面有个``SparkSubmitArguments``方法，这个肯定就是匹配spark任务提交时候的参数了。就是你写到你的sparkConf里面的东西

再看submit方法：

```scala
private def submit(args: SparkSubmitArguments): Unit = {
	//这个解析的方法要重点看
    val (childArgs, childClasspath, sysProps, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      //····
      } else {
        runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
      }
    }
  if (args.isStandaloneCluster && args.useRest) {
      try {
       //····
        doRunMain()
      } catch {
        //····
          submit(args)
      }
    // In all other modes, just run the main class as prepared
    } else {
      doRunMain()
    }
```
反正最后都是要执行``runMain``方法，先来看我上面说的重点要看的那个方法``prepareSubmitEnvironment``

```scala
val childArgs = new ArrayBuffer[String]()
    val childClasspath = new ArrayBuffer[String]()
    val sysProps = new HashMap[String, String]()
    var childMainClass = ""
 val clusterManager: Int = args.master match {
      case m if m.startsWith("yarn") => YARN
      case m if m.startsWith("spark") => STANDALONE
      case m if m.startsWith("mesos") => MESOS
      case m if m.startsWith("local") => LOCAL
      case _ => printErrorAndExit("Master must start with yarn, spark, mesos, or local"); -1
    }
	//.....
	if (args.useRest) {
        childMainClass = "org.apache.spark.deploy.rest.RestSubmissionClient"
	} else {
        // In legacy standalone cluster mode, use Client as a wrapper around the user class
        childMainClass = "org.apache.spark.deploy.Client"
	if (isYarnCluster) {
      childMainClass = "org.apache.spark.deploy.yarn.Client"
```
我主要是想说的这个childMainClass,这里的MainClass不是你写的spark任务的里面的你自己的那个class，而是spark自己写的，``RestSubmissionClient，deploy.Client，yarn.Client``是这三个中的一个，为什么这里主要强调这个地方，下面继续说。

runMain方法：

```scala
 private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sysProps: Map[String, String],
      childMainClass: String,
      verbose: Boolean): Unit = {
    // scalastyle:off println
    if (verbose) {
      printStream.println(s"Main class:\n$childMainClass")
      printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")
      printStream.println(s"System properties:\n${sysProps.mkString("\n")}")
      printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      printStream.println("\n")
    }
    // scalastyle:on println

//....
    var mainClass: Class[_] = null

    try {
      mainClass = Utils.classForName(childMainClass)
  
    val mainMethod = mainClass.getMethod("main", new Array[String]
      mainMethod.invoke(null, childArgs.toArray)
```

看到没，最终是反射了``childMainClass``然后执行了里面的Main方法，***不是你自己写的那个main***方法，为啥我要强调这个，因为我也看了网上有人写spark-submit任务提交的源码分析，我擦，他居然写的是这个invoke执行的是自己写的main方法，这简直不负责任了吧，不多说了，谷歌一收就能收出来，排在最前面的，给个链接好了：http://www.jianshu.com/p/0b19b2b954b6  认真看，你就能发现，谷歌排在前面的，前几篇都是一样，互相抄。

后面就是该提交任务了，你用哪种方式提交就执行哪个``main``方法，``client``看了代码比``yarn-cluster``少，``yarn-cluster``就是正式提交任务给yarn了，提交任务之后都会分配一个applicationId.....后面的请听下回分解