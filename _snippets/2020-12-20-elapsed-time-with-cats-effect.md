---
tags:
  - Cats Effect
  - FP
  - Scala
---
I wasn't really satisfied with the answers [posted][1] on SO.
```scala
import cats.effect.{ ExitCode, IO, IOApp }

import scala.concurrent.duration.MILLISECONDS

object Elapsed extends IOApp {

  override def run(args: List[String]): IO[ExitCode] =
    for {
      _ <- timer.clock
        .monotonic(MILLISECONDS)
        .bracket { _ =>
          ...
      } { start =>
        timer.clock
          .monotonic(MILLISECONDS)
          .flatMap(end => ...)
      }
    } yield ExitCode.Success
}
```

[1]: https://stackoverflow.com/a/54832070