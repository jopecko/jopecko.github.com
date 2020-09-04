---
layout: post
title:  "Auto Derived Typeclasses - a cautionary tale"
date: 2020-08-26 20:26:27 GMT-7
categories: scala circe typeclasses performance
description: In which auto derived serdes took down our CI build
---

On a recent greenfield project, where we were quickly iterating on features and hammering
out the shape of a new Scala &mu;service, we decided to use Circe's
[automatic typeclass derivation][1] to synthesize the `Encoder`s and `Decoder`s for the
GADTs used in our REST layer. At first glance, automatic derivation seemed like a total
win. The codebase was clean and unencumbered with any of the necessary boilerplate usually
involving the declaration of serdes. The compiler will materialize any typeclass type `T`
that's not in scope. All that's required is a simple import `io.circe.generic.auto._`.

At first everything was great as we were free to hack with wild abandon as we fleshed out
the requirements. In retrospect it feels that relying so heavily on this was willfully
embracing [creeping normality][2] as compilation times starting ticking slowly, imperceptibly,
upwards. Looking back, it should have been obvious there was issues looming in the future.
However, these subtle increases to the project's compilation time is pratically undetectable
and too easily ignored. Scala is already quite well-known for lengthy compilation times and
during a typical day, when you're busy hacking, compiling, reading slack, etc., its all too
easy to neglect.

The first sign of trouble appeared seemingly out of no where. An otherwise standard and
minimally additive patch started behaving erratically on our CI system. Builds were successful
*most* of the time, but occassionally failed. And when they failed, they failed for baffling
reasons. Coupled with the fact that we had recently upgraded `sbt`, we found ourselves chasing
non-existent tooling issues. For example, this was an example of one the interrmitent errors we
were seeing:

```
[info] Compiling 75 Scala sources to /some-service/target/scala-2.13/classes ...
[error] ## Exception when compiling 75 sources to /some-service/target/scala-2.13/classes
[error] java.lang.NoClassDefFoundError: xsbt/DelegatingReporter$CompileProblem
[error] xsbt.DelegatingReporter.info0(DelegatingReporter.scala:181)
[error] scala.tools.nsc.reporters.MakeFilteringForwardingReporter.doReport(ForwardingReporter.scala:59)
[error] scala.tools.nsc.reporters.FilteringReporter.info0(Reporter.scala:93)
[error] scala.reflect.internal.Reporter.echo(Reporting.scala:112)
[error] scala.reflect.internal.Reporter.echo(Reporting.scala:111)
[error] scoverage.ScoverageInstrumentationComponent$$anon$1.run(plugin.scala:119)
[error] scala.tools.nsc.Global$Run.compileUnitsInternal(Global.scala:1505)
[error] scala.tools.nsc.Global$Run.compileUnits(Global.scala:1489)
[error] scala.tools.nsc.Global$Run.compileSources(Global.scala:1481)
[error] scala.tools.nsc.Global$Run.compile(Global.scala:1616)
[error] xsbt.CachedCompiler0.run(CompilerInterface.scala:153)
[error] xsbt.CachedCompiler0.run(CompilerInterface.scala:125)
[error] xsbt.CompilerInterface.run(CompilerInterface.scala:39)
[error] java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
[error] java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
[error] java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
[error] java.base/java.lang.reflect.Method.invoke(Method.java:566)
[error] sbt.internal.inc.AnalyzingCompiler.call(AnalyzingCompiler.scala:248)
[error] sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:122)
[error] sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:95)
[error] sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$4(MixedAnalyzingCompiler.scala:91)
[error] scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
[error] sbt.internal.inc.MixedAnalyzingCompiler.timed(MixedAnalyzingCompiler.scala:186)
[error] sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3(MixedAnalyzingCompiler.scala:82)
[error] sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3$adapted(MixedAnalyzingCompiler.scala:77)
[error] sbt.internal.inc.JarUtils$.withPreviousJar(JarUtils.scala:215)
[error] sbt.internal.inc.MixedAnalyzingCompiler.compileScala$1(MixedAnalyzingCompiler.scala:77)
[error] sbt.internal.inc.MixedAnalyzingCompiler.compile(MixedAnalyzingCompiler.scala:146)
[error] sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1(IncrementalCompilerImpl.scala:343)
[error] sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1$adapted(IncrementalCompilerImpl.scala:343)
[error] sbt.internal.inc.Incremental$.doCompile(Incremental.scala:120)
[error] sbt.internal.inc.Incremental$.$anonfun$compile$4(Incremental.scala:100)
[error] sbt.internal.inc.IncrementalCommon.recompileClasses(IncrementalCommon.scala:180)
[error] sbt.internal.inc.IncrementalCommon.cycle(IncrementalCommon.scala:98)
[error] sbt.internal.inc.Incremental$.$anonfun$compile$3(Incremental.scala:102)
[error] sbt.internal.inc.Incremental$.manageClassfiles(Incremental.scala:155)
[error] sbt.internal.inc.Incremental$.compile(Incremental.scala:92)
[error] sbt.internal.inc.IncrementalCompile$.apply(Compile.scala:75)
[error] sbt.internal.inc.IncrementalCompilerImpl.compileInternal(IncrementalCompilerImpl.scala:348)
[error] sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileIncrementally$1(IncrementalCompilerImpl.scala:301)
[error] sbt.internal.inc.IncrementalCompilerImpl.handleCompilationError(IncrementalCompilerImpl.scala:168)
[error] sbt.internal.inc.IncrementalCompilerImpl.compileIncrementally(IncrementalCompilerImpl.scala:248)
[error] sbt.internal.inc.IncrementalCompilerImpl.compile(IncrementalCompilerImpl.scala:74)
[error] sbt.Defaults$.compileIncrementalTaskImpl(Defaults.scala:1765)
[error] sbt.Defaults$.$anonfun$compileIncrementalTask$1(Defaults.scala:1738)
[error] scala.Function1.$anonfun$compose$1(Function1.scala:49)
[error] sbt.internal.util.$tilde$greater.$anonfun$$u2219$1(TypeFunctions.scala:62)
[error] sbt.std.Transform$$anon$4.work(Transform.scala:67)
[error] sbt.Execute.$anonfun$submit$2(Execute.scala:281)
[error] sbt.internal.util.ErrorHandling$.wideConvert(ErrorHandling.scala:19)
[error] sbt.Execute.work(Execute.scala:290)
[error] sbt.Execute.$anonfun$submit$1(Execute.scala:281)
[error] sbt.ConcurrentRestrictions$$anon$4.$anonfun$submitValid$1(ConcurrentRestrictions.scala:178)
[error] sbt.CompletionService$$anon$2.call(CompletionService.scala:37)
[error] java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
[error] java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
[error] java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
[error] java.base/java.lang.Thread.run(Thread.java:834)
[error]
[error] java.lang.NoClassDefFoundError: xsbt/DelegatingReporter$CompileProblem
[error] 	at xsbt.DelegatingReporter.info0(DelegatingReporter.scala:181)
[error] 	at scala.tools.nsc.reporters.MakeFilteringForwardingReporter.doReport(ForwardingReporter.scala:59)
[error] 	at scala.tools.nsc.reporters.FilteringReporter.info0(Reporter.scala:93)
[error] 	at scala.reflect.internal.Reporter.echo(Reporting.scala:112)
[error] 	at scala.reflect.internal.Reporter.echo(Reporting.scala:111)
[error] 	at scoverage.ScoverageInstrumentationComponent$$anon$1.run(plugin.scala:119)
[error] 	at scala.tools.nsc.Global$Run.compileUnitsInternal(Global.scala:1505)
[error] 	at scala.tools.nsc.Global$Run.compileUnits(Global.scala:1489)
[error] 	at scala.tools.nsc.Global$Run.compileSources(Global.scala:1481)
[error] 	at scala.tools.nsc.Global$Run.compile(Global.scala:1616)
[error] 	at xsbt.CachedCompiler0.run(CompilerInterface.scala:153)
[error] 	at xsbt.CachedCompiler0.run(CompilerInterface.scala:125)
[error] 	at xsbt.CompilerInterface.run(CompilerInterface.scala:39)
[error] 	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
[error] 	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
[error] 	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
[error] 	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
[error] 	at sbt.internal.inc.AnalyzingCompiler.call(AnalyzingCompiler.scala:248)
[error] 	at sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:122)
[error] 	at sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:95)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$4(MixedAnalyzingCompiler.scala:91)
[error] 	at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.timed(MixedAnalyzingCompiler.scala:186)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3(MixedAnalyzingCompiler.scala:82)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3$adapted(MixedAnalyzingCompiler.scala:77)
[error] 	at sbt.internal.inc.JarUtils$.withPreviousJar(JarUtils.scala:215)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.compileScala$1(MixedAnalyzingCompiler.scala:77)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.compile(MixedAnalyzingCompiler.scala:146)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1(IncrementalCompilerImpl.scala:343)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1$adapted(IncrementalCompilerImpl.scala:343)
[error] 	at sbt.internal.inc.Incremental$.doCompile(Incremental.scala:120)
[error] 	at sbt.internal.inc.Incremental$.$anonfun$compile$4(Incremental.scala:100)
[error] 	at sbt.internal.inc.IncrementalCommon.recompileClasses(IncrementalCommon.scala:180)
[error] 	at sbt.internal.inc.IncrementalCommon.cycle(IncrementalCommon.scala:98)
[error] 	at sbt.internal.inc.Incremental$.$anonfun$compile$3(Incremental.scala:102)
[error] 	at sbt.internal.inc.Incremental$.manageClassfiles(Incremental.scala:155)
[error] 	at sbt.internal.inc.Incremental$.compile(Incremental.scala:92)
[error] 	at sbt.internal.inc.IncrementalCompile$.apply(Compile.scala:75)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compileInternal(IncrementalCompilerImpl.scala:348)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileIncrementally$1(IncrementalCompilerImpl.scala:301)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.handleCompilationError(IncrementalCompilerImpl.scala:168)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compileIncrementally(IncrementalCompilerImpl.scala:248)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compile(IncrementalCompilerImpl.scala:74)
[error] 	at sbt.Defaults$.compileIncrementalTaskImpl(Defaults.scala:1765)
[error] 	at sbt.Defaults$.$anonfun$compileIncrementalTask$1(Defaults.scala:1738)
[error] 	at scala.Function1.$anonfun$compose$1(Function1.scala:49)
[error] 	at sbt.internal.util.$tilde$greater.$anonfun$$u2219$1(TypeFunctions.scala:62)
[error] 	at sbt.std.Transform$$anon$4.work(Transform.scala:67)
[error] 	at sbt.Execute.$anonfun$submit$2(Execute.scala:281)
[error] 	at sbt.internal.util.ErrorHandling$.wideConvert(ErrorHandling.scala:19)
[error] 	at sbt.Execute.work(Execute.scala:290)
[error] 	at sbt.Execute.$anonfun$submit$1(Execute.scala:281)
[error] 	at sbt.ConcurrentRestrictions$$anon$4.$anonfun$submitValid$1(ConcurrentRestrictions.scala:178)
[error] 	at sbt.CompletionService$$anon$2.call(CompletionService.scala:37)
[error] 	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] 	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
[error] 	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] 	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
[error] 	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
[error] 	at java.base/java.lang.Thread.run(Thread.java:834)
[error] Caused by: java.lang.ClassNotFoundException: xsbt.DelegatingReporter$CompileProblem
[error] 	at java.base/java.net.URLClassLoader.findClass(URLClassLoader.java:471)
[error] 	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:588)
[error] 	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521)
[error] 	at xsbt.DelegatingReporter.info0(DelegatingReporter.scala:181)
[error] 	at scala.tools.nsc.reporters.MakeFilteringForwardingReporter.doReport(ForwardingReporter.scala:59)
[error] 	at scala.tools.nsc.reporters.FilteringReporter.info0(Reporter.scala:93)
[error] 	at scala.reflect.internal.Reporter.echo(Reporting.scala:112)
[error] 	at scala.reflect.internal.Reporter.echo(Reporting.scala:111)
[error] 	at scoverage.ScoverageInstrumentationComponent$$anon$1.run(plugin.scala:119)
[error] 	at scala.tools.nsc.Global$Run.compileUnitsInternal(Global.scala:1505)
[error] 	at scala.tools.nsc.Global$Run.compileUnits(Global.scala:1489)
[error] 	at scala.tools.nsc.Global$Run.compileSources(Global.scala:1481)
[error] 	at scala.tools.nsc.Global$Run.compile(Global.scala:1616)
[error] 	at xsbt.CachedCompiler0.run(CompilerInterface.scala:153)
[error] 	at xsbt.CachedCompiler0.run(CompilerInterface.scala:125)
[error] 	at xsbt.CompilerInterface.run(CompilerInterface.scala:39)
[error] 	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
[error] 	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
[error] 	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
[error] 	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
[error] 	at sbt.internal.inc.AnalyzingCompiler.call(AnalyzingCompiler.scala:248)
[error] 	at sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:122)
[error] 	at sbt.internal.inc.AnalyzingCompiler.compile(AnalyzingCompiler.scala:95)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$4(MixedAnalyzingCompiler.scala:91)
[error] 	at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.timed(MixedAnalyzingCompiler.scala:186)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3(MixedAnalyzingCompiler.scala:82)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.$anonfun$compile$3$adapted(MixedAnalyzingCompiler.scala:77)
[error] 	at sbt.internal.inc.JarUtils$.withPreviousJar(JarUtils.scala:215)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.compileScala$1(MixedAnalyzingCompiler.scala:77)
[error] 	at sbt.internal.inc.MixedAnalyzingCompiler.compile(MixedAnalyzingCompiler.scala:146)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1(IncrementalCompilerImpl.scala:343)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileInternal$1$adapted(IncrementalCompilerImpl.scala:343)
[error] 	at sbt.internal.inc.Incremental$.doCompile(Incremental.scala:120)
[error] 	at sbt.internal.inc.Incremental$.$anonfun$compile$4(Incremental.scala:100)
[error] 	at sbt.internal.inc.IncrementalCommon.recompileClasses(IncrementalCommon.scala:180)
[error] 	at sbt.internal.inc.IncrementalCommon.cycle(IncrementalCommon.scala:98)
[error] 	at sbt.internal.inc.Incremental$.$anonfun$compile$3(Incremental.scala:102)
[error] 	at sbt.internal.inc.Incremental$.manageClassfiles(Incremental.scala:155)
[error] 	at sbt.internal.inc.Incremental$.compile(Incremental.scala:92)
[error] 	at sbt.internal.inc.IncrementalCompile$.apply(Compile.scala:75)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compileInternal(IncrementalCompilerImpl.scala:348)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.$anonfun$compileIncrementally$1(IncrementalCompilerImpl.scala:301)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.handleCompilationError(IncrementalCompilerImpl.scala:168)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compileIncrementally(IncrementalCompilerImpl.scala:248)
[error] 	at sbt.internal.inc.IncrementalCompilerImpl.compile(IncrementalCompilerImpl.scala:74)
[error] 	at sbt.Defaults$.compileIncrementalTaskImpl(Defaults.scala:1765)
[error] 	at sbt.Defaults$.$anonfun$compileIncrementalTask$1(Defaults.scala:1738)
[error] 	at scala.Function1.$anonfun$compose$1(Function1.scala:49)
[error] 	at sbt.internal.util.$tilde$greater.$anonfun$$u2219$1(TypeFunctions.scala:62)
[error] 	at sbt.std.Transform$$anon$4.work(Transform.scala:67)
[error] 	at sbt.Execute.$anonfun$submit$2(Execute.scala:281)
[error] 	at sbt.internal.util.ErrorHandling$.wideConvert(ErrorHandling.scala:19)
[error] 	at sbt.Execute.work(Execute.scala:290)
[error] 	at sbt.Execute.$anonfun$submit$1(Execute.scala:281)
[error] 	at sbt.ConcurrentRestrictions$$anon$4.$anonfun$submitValid$1(ConcurrentRestrictions.scala:178)
[error] 	at sbt.CompletionService$$anon$2.call(CompletionService.scala:37)
[error] 	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] 	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
[error] 	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
[error] 	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
[error] 	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
[error] 	at java.base/java.lang.Thread.run(Thread.java:834)
[error] (Compile / compileIncremental) java.lang.NoClassDefFoundError: xsbt/DelegatingReporter$CompileProblem
[error] Total time: 115 s (01:55), completed Aug 21, 2020, 10:37:21 PM
```

Upon stumbling on this error, we started to think we had some incompatibilities with the
compiler plugins we were using and the version of `sbt`. Every plugin backed out and `sbt`
version change produced the same or similar failures. Some builds succeeded and some failed.
 We had to start looking elsewhere.

Since we build within a container, we started to suspect that perhaps we were running out of
memory in our build process. We had to allocate an obsene amount of memory to our container
to get the build back to consistently green. This was when I started to suspect a problem
with our implicit approach.

Running builds with the following [compiler flags][3]:
* `-Ystatistics`
* `-Yshow-trees`
* `-Ydebug`

started to shed some light on the situation. The times spent handling implicits seemed to have
become an issue. Then I discovered [this][4] blog post on scalac profiling, which discusses the
cost of implicit macros. At this point I figured a quick fix would be to just switch to Circe's
[semi-automatic derivation][5] instead. Placing them in the companion objects for the GADTs
minimizes the [implicit import tax][7] as well.

In retrospect, I feel like my experience should have warned me off of auto derivation but the
appeal was too great at the time. I can only hope that, given another chance to make a similar
decision, I don't repeat this mistake. I'm sure auto derived typeclasses have their place,
however, I'm not sure at this time what that could be. Time permitting, I'd like to revisit this
in the future and do a proper post-mortem on this comparing the two approaches and figuring out
what was saved by changing from one approach to the other. For another time though...


[1]: https://circe.github.io/circe/codecs/auto-derivation.html
[2]: https://en.wikipedia.org/wiki/Creeping_normality
[3]: https://docs.scala-lang.org/overviews/compiler-options/index.html
[4]: https://www.scala-lang.org/blog/2018/06/04/scalac-profiling.html
[5]: https://circe.github.io/circe/codecs/semiauto-derivation.html
[6]: https://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html
[7]: https://vimeo.com/20308847