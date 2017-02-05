<!DOCTYPE html>
<html>
  <head>
    <title>Title</title>
    <meta charset="utf-8">
    <style>
      @import url(https://fonts.googleapis.com/css?family=Yanone+Kaffeesatz);
      @import url(https://fonts.googleapis.com/css?family=Droid+Serif:400,700,400italic);
      @import url(https://fonts.googleapis.com/css?family=Ubuntu+Mono:400,700,400italic);

      body { font-family: 'Droid Serif'; }
      h1, h2, h3 {
        font-family: 'Yanone Kaffeesatz';
        font-weight: normal;
      }
      .remark-code, .remark-inline-code { font-family: 'Ubuntu Mono'; }
    </style>
  </head>
  <body>
    <textarea id="source">

# Scala Resource Management

## IndyScala, February 6, 2017

```tut:invisible
import java.io._

def bufferedReader(filename: String) = {
  val in = getClass.getResourceAsStream(filename)
  val r = new InputStreamReader(in)
  new BufferedReader(r)
}
```

---

# wc -l, à la Java

```tut:book
def countLines: Int = {
  val br = bufferedReader("/fahrenheit.txt")
  var lc = 0
  try {
    var line: String = null
    while ({ line = br.readLine(); line != null}) {
      lc += 1
    }
  } 
  finally br.close()
  return lc
}
countLines
```

---

# Return, schmeturn

```tut:book
def countLines: Int = {
  val br = bufferedReader("/fahrenheit.txt")
  var lc = 0
  try {
    var line: String = null
    while ({ line = br.readLine(); line != null}) {
      lc += 1
    }
  } 
  finally br.close()
  lc
}
countLines
```

---

# try-catch-finally is an expression

```tut:book
def countLines = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    var lc = 0
    var line: String = null
    while ({ line = br.readLine(); line != null}) {
      lc += 1
    }
    lc
  } 
  finally br.close()
}
countLines
```

---

# Recursion

```tut:book
def countLines = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    def loop(lc: Int): Int =
      br.readLine() match {
        case s: String => loop(lc + 1)
        case null => lc
      }
    loop(0)
  } 
  finally br.close()
}
countLines
```

---

# Streaming

```tut:book
def countLines = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    Stream.continually(br.readLine())
      .takeWhile(_ != null)
      .size
  } 
  finally br.close()
}
countLines
```

---

# Fahrenheit to Celsius, à la Java

```tut:book
def fToC(f: Double) = (f - 32.0) * 5.0 / 9.0

def convert() = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    val out = new FileWriter("/tmp/celsius.txt")
    try {
      var line: String = null
      while ({ line = br.readLine(); line != null}) {
        if (!line.startsWith("//")) {
          val f = line.toDouble
          val c = fToC(f)
          out.write(c.toString)
          out.write("\n")
        }
      }
      out.flush()
    } 
    finally out.close()
  } 
  finally br.close()
}
convert()
```

---

# Fahrenheit to Celsius, Streaming

```tut:book
def convert() = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    val out = new FileWriter("/tmp/celsius.txt")
    try {
      Stream.continually(br.readLine())
        .takeWhile(_ != null)
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
    finally out.close()
  } 
  finally br.close()
}
convert()
```

---

# Boilerplate

```tut:book
def boilerplate[A]: A = {
  val br = bufferedReader("/fahrenheit.txt")
  try {
    ??? // Something that returns an A
  } 
  finally br.close()
}
```

---

# withBufferedReader

```tut:book
def withBufferedReader[A](br: => BufferedReader)(f: BufferedReader => A): A = {
  try f(br)
  finally { println("Closing "+br); br.close() }
}

def boilerplate[A]: A = {
  withBufferedReader(bufferedReader("/fahrenheit.txt")) { br =>
    ???
  }
}
```

---

# wc -l, with buffered reader

```tut:book
def countLines =
  withBufferedReader(bufferedReader("/fahrenheit.txt")) { br =>
    Stream.continually(br.readLine())
      .takeWhile(_ != null)
      .size
  } 
countLines
```

---

# F to C, with buffered reader

```tut:book
def convert() =
  withBufferedReader(bufferedReader("/fahrenheit.txt")) { br =>
    val out = new FileWriter("/tmp/celsius.txt")
    try {
      Stream.continually(br.readLine())
        .takeWhile(_ != null)
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
    finally out.close()
  } 

convert()
```

---

# We can do the same for writers

```tut:book
def withFileWriter[A](fw: => FileWriter)(f: FileWriter => A) = {
  try f(fw)
  finally { println("Closing "+fw); fw.close() }
}
def convert() =
  withBufferedReader(bufferedReader("/fahrenheit.txt")) { br =>
    withFileWriter(new FileWriter("/tmp/celsius.txt")) { out =>
      Stream.continually(br.readLine())
        .takeWhile(_ != null)
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  } 
convert()
```

---

# ...or anything closeable!

```tut:book
def withResource[R <: Closeable, A](r: => R)(f: R => A) = {
  try f(r)
  finally { println("Closing "+r); r.close() }
}
def convert() =
  withResource(bufferedReader("/fahrenheit.txt")) { br =>
    withResource(new FileWriter("/tmp/celsius.txt")) { out =>
      Stream.continually(br.readLine())
        .takeWhile(_ != null)
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  } 
convert()
```

---

# An iterator of Strings

```tut:silent
trait Source[+A] extends Iterator[A] {
  def close(): Unit
}
object Source {
  def lines(filename: String): Source[String] = 
    new Source[String] {
      private val br = bufferedReader(filename)
      private val it =
        Stream.continually(br.readLine()).takeWhile(_ != null).toIterator
      def hasNext: Boolean = it.hasNext
      def next(): String = it.next()
      def close() = br.close()
      override def toString = s"Source.lines($filename)"
    }
}
```

---

# But it's not Closeable

```tut:book:fail
def convert() = {
  withResource(Source.lines("/fahrenheit.txt")) { source =>
    withResource(new FileWriter("/tmp/celsius.txt")) { out =>
      source
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  }
}
```

---

# What we're about to do is a bad idea

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Succinct and accurate. <a href="http://t.co/Z2Gd2xOB8g">pic.twitter.com/Z2Gd2xOB8g</a></p>&mdash; Matt Mastracci (@mmastrac) <a href="https://twitter.com/mmastrac/status/536332443398057984">November 23, 2014</a></blockquote>

---

# Duck (aka "structural") typing

```tut:book
def withResource[R <: { def close(): Unit }, A](r: => R)(f: R => A) = {
  try f(r)
  finally { println("Closing "+r); r.close() }
}
def convert() = {
  withResource(Source.lines("/fahrenheit.txt")) { source =>
    withResource(new FileWriter("/tmp/celsius.txt")) { out =>
      source
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  }
}
convert()
```

---

# This one quacks like a goose

```tut:book
trait HipsterSource[+A] extends Iterator[A] {
  def dispose(): Unit
}
object HipsterSource {
  def lines(filename: String): HipsterSource[String] = 
    new HipsterSource[String] {
      private val br = bufferedReader(filename)
      private val it =
        Stream.continually(br.readLine()).takeWhile(_ != null).toIterator
      def hasNext: Boolean = it.hasNext
      def next(): String = it.next()
      def dispose() = br.close()
      override def toString = s"HipsterSource.lines($filename)"
    }
}
```

---

# Nope

```tut:book:fail
def convert() = {
  withResource(HipsterSource.lines("/fahrenheit.txt")) { source =>
    withResource(new FileWriter("/tmp/celsius.txt")) { out =>
      source
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  }
}
 ```

---

# Typeclasses

```tut:book
trait Resource[R] {
  def close(r: R): Unit
}

object Resource {
  implicit def CloseableResource[R <: Closeable]: Resource[R] =
    new Resource[R] {
      def close(r: R) = {
        println("Closing "+r); r.close()
      }
    }
  implicit def HipsterSourceResource[A]: Resource[HipsterSource[A]] =
    new Resource[HipsterSource[A]] {
      def close(r: HipsterSource[A]) = {
        println("Disposing "+r); r.dispose()
      }
    }
}

def withResource[R: Resource, A](r: => R)(f: R => A) = {
  try f(r)
  finally implicitly[Resource[R]].close(r)
}
```
```tut:invisible
import Resource._ // work around REPL deficiency
```

---

# A disposable example

```tut:book
def convert() = {
  withResource(HipsterSource.lines("/fahrenheit.txt")) { source =>
    withResource(new FileWriter("/tmp/celsius.txt")) { out =>
      source
        .filterNot(_.startsWith("//"))
        .map(_.toDouble)
        .map(fToC(_))
        .foreach { c =>
          out.write(c.toString)
          out.write("\n")
        }
      out.flush()
    } 
  }
}
convert()
```

---

# This is the longest Ross has ever talked without talking about monads

<center>
<img src="http://www.reactiongifs.com/wp-content/uploads/2012/11/tumblr_llo6cv7fck1qfiwl0.gif"/>
</center>

---

# The Managed monad

```tut:book
trait Managed[R] { self =>
  def attemptWith[A](f: R => A): Either[Throwable, A]

  def flatMap[B](f: R => Managed[B]): Managed[B] =
    new Managed[B] {
      def attemptWith[C](g: B => C): Either[Throwable, C] = {
        self.attemptWith(r => f(r).attemptWith(g)).fold(e => Left(e), identity)
      }
    }
    
  def map[B](f: R => B): Managed[B] =
    new Managed[B] {
      def attemptWith[C](g: B => C): Either[Throwable, C] = {
        self.attemptWith(r => f(r)).fold(e => Left(e), b => Right(g(b)))
      }
    }

  def foreach(f: R => Unit): Unit =
    attemptWith(f).fold(throw _, _ => ())
}
```

---

# And some ways to construct it

```tut:book
object Managed {
  def bracket[R](acquire: => R)(close: R => Unit): Managed[R] =
    new Managed[R] {
      def attemptWith[A](f: R => A) = {
        val r = acquire
        try Right(f(r)) 
        catch { case e: Throwable => Left(e) }
        finally close(r)
      }
    }
  
  def apply[R: Resource](acquire: => R): Managed[R] =
    bracket(acquire)(implicitly[Resource[R]].close)
  
  // It's not really a monad without a point
  def point[R](r: => R): Managed[R] =
    bracket(r)(_ => {})
}
```

---

# Flatmap your resources

```tut:book
def convert() = {
  for {
    source <- Managed(HipsterSource.lines("/fahrenheit.txt"))
    out    <- Managed(new FileWriter("/tmp/celsius.txt"))
  } {
    source
      .filterNot(_.startsWith("//"))
      .map(_.toDouble)
      .map(fToC(_))
      .foreach { c =>
        out.write(c.toString)
        out.write("\n")
      }
    out.flush()
  }
}
convert()
```

---

# This is a poor man's scala-arm

<center>
<a href="https://github.com/jsuereth/scala-arm">
<img src="https://shrinktheweb.snapito.io/v2/webshot/spu-ea68c8-ogi2-3cwn3bmfojjlb56e?size=800x0&screen=1024x768&url=http%3A%2F%2Fgithub.com%2Fjsuereth%2Fscala-arm" />
</a>
</center>

---

# That was fun, but not functional

```tut:book
import fs2.Task

trait FResource[R] {
  def close(r: R): Task[Unit]
}

def withFResource[R: FResource, A](acquire: => R)(f: R => Task[A]) = for {
  r <- Task.delay(acquire)
  att <- f(r).attempt
  _ <- implicitly[FResource[R]].close(r).attempt
  a <- att.fold(Task.fail, Task.now)
} yield a

```
---

# Provide some instances

```tut:book
object FResource {
  implicit def closeableFResource[R <: Closeable]: FResource[R] =
    new FResource[R] {
      def close(r: R) = Task.delay {
        println("Closing "+r); r.close()
      }
    }
  implicit def hipsterSourceFResource[A]: FResource[HipsterSource[A]] =
    new FResource[HipsterSource[A]] {
      def close(r: HipsterSource[A]) = Task.delay {
        println("Disposing "+r); r.dispose()
      }
    }
}
```
```tut:invisible
import FResource._ // work around REPL deficiency
```

---

# Bundle it all into a Task

```tut:book
def convert = {
  withFResource(HipsterSource.lines("/fahrenheit.txt")) { source =>
    withFResource(new FileWriter("/tmp/celsius.txt")) { out =>
      Task.delay {
        source
          .filterNot(_.startsWith("//"))
          .map(_.toDouble)
          .map(fToC(_))
          .foreach { c =>
            out.write(c.toString)
            out.write("\n")
          }
        out.flush()
      }
    } 
  }
}
convert
convert.unsafeRun
```

---

# Works with Managed, too

```tut:book
import fs2.Task

trait FManaged[R] { self =>
  def withResource[A](f: R => Task[A]): Task[A]

  def flatMap[B](f: R => FManaged[B]): FManaged[B] =
    new FManaged[B] {
      def withResource[C](g: B => Task[C]): Task[C] = {
        self.withResource(r => f(r).withResource(g))
      }
    }
    
  def map[B](f: R => B): FManaged[B] =
    new FManaged[B] {
      def withResource[C](g: B => Task[C]): Task[C] = {
        self.withResource(r => g(f(r)))
      }
    }

  def run: Task[R] = 
    withResource(r => Task.delay(r))
}
```

---

```tut:book
object FManaged {
  def bracket[R](acquire: Task[R])(close: R => Task[Unit]): FManaged[R] =
    new FManaged[R] {
      def withResource[A](f: R => Task[A]) = for {
        r <- acquire
        att <- f(r).attempt
        _ <- close(r).attempt
        a <- att.fold(Task.fail, Task.now)
      } yield a
    }
  
  def apply[R: FResource](r: => R): FManaged[R] =
    bracket(Task.delay(r))(implicitly[FResource[R]].close)
  
  // It's not really a monad without a point
  def point[R](r: => R): FManaged[R] =
    bracket(Task.delay(r))(_ => Task.now(()))
}
```

---

```tut:book
def convert = {
  for {
    source <- FManaged(HipsterSource.lines("/fahrenheit.txt"))
    out    <- FManaged(new FileWriter("/tmp/celsius.txt"))
  } yield {
    source
      .filterNot(_.startsWith("//"))
      .map(_.toDouble)
      .map(fToC(_))
      .foreach { c =>
        out.write(c.toString)
        out.write("\n")
      }
    out.flush()
  }
}
convert.run
convert.run.unsafeRun
```

---

# FManaged is called Codensity in Scalaz

```tut:book
import scalaz.Codensity
import fs2.interop.scalaz._

def codensity[R: FResource](r: => R): Codensity[Task, R] =
  new Codensity[Task, R] {
    def apply[A](f: R => Task[A]): Task[A] =
      withFResource(r)(f)
   }
```

---

# And run is called improve

```tut:book
def convert = {
  for {
    source <- codensity(HipsterSource.lines("/fahrenheit.txt"))
    out    <- codensity(new FileWriter("/tmp/celsius.txt"))
  } yield {
    source
      .filterNot(_.startsWith("//"))
      .map(_.toDouble)
      .map(fToC(_))
      .foreach { c =>
        out.write(c.toString)
        out.write("\n")
      }
    out.flush()
  }
}
convert.improve
convert.improve.unsafeRun
```

---

# Beware of interruptions

```tut:book
import fs2.{Strategy, Scheduler}
import scala.concurrent.duration._

implicit val S = Strategy.fromFixedDaemonPool(4)
implicit val scheduler = Scheduler.fromFixedDaemonPool(1)

convert.improve.unsafeRun
convert.improve.unsafeTimed(1.nanosecond).attempt.unsafeRun
```

[Scalaz is trying to fix this.](https://github.com/scalaz/scalaz/pull/1317#pullrequestreview-19145654)

---

# FS2 Streams are good at resource management

```tut:book
import fs2.Stream
import java.util.UUID

val acquire = Task.delay { val id = UUID.randomUUID; println("Acquiring "+id); id }
def release(id: UUID) = Task.delay { println("Releasing "+id) }
def use(id: UUID) = Stream.eval(Task.delay { println("I did something with "+id) })
val s = Stream.bracket(acquire)(use, release)
s.run.unsafeRun
```

---

# And a lot of what we do with resources is Stream

```tut:book
import fs2.{io, text}
import java.nio.file.Paths
val convert: Task[Unit] = {
  val in = Task.delay(getClass.getResourceAsStream("/fahrenheit.txt"))
  io.readInputStream[Task](in, 4096)
    .through(text.utf8Decode)
    .through(text.lines)
    .filter(s => !s.trim.isEmpty && !s.startsWith("//"))
    .map(line => fToC(line.toDouble).toString)
    .intersperse("\n")
    .through(text.utf8Encode)
    .through(io.file.writeAll(Paths.get("/tmp/celsius.txt")))
    .run
}
convert.unsafeRun()
```

...but FS2 is a whole other presentation.

---

    </Textarea>
    <script src="https://gnab.github.io/remark/downloads/remark-latest.min.js">
    </script>
    <script>
      var slideshow = remark.create();
    </script>
    <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
  </body>
</html>
