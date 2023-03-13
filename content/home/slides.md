+++
weight = 1
+++

## Side effect nei linguaggi di programmazione

> If a program has no __side effects__ there’s no point in running it.
> You have a black box, you press go and it gets hot but there’s no output
> _Simon Peyton-Jones_

```scala
def appendToFile(file: String, line: String): Unit =
  println(f"Appending $line to $file")
  Using(FileWriter(file, true))(_.write(f"$line\n"))
```

---

> __Side effects are lies.__ Your function promises to do one thing, but it also does other hidden things.
> [...] They are mistruths that often result in strange temporal couplings and order dependencies.
> _Robert Cecil Martin_

{{% multicol %}}

{{% col %}}

```scala
// f : Double => Double
def f(x: Double): Double =
  appendToFile("path/to/file", "Effect!")
  Math.cos(x)

f(x) + f(x) =/= 2 * f(x)
```

{{% /col %}}

{{% col %}}

```scala
// g : Double => Double
def g(x: Double): Double =
  Math.cos(x)


g(x) + g(x) === 2 * g(x)
```

{{% /col %}}

{{% /multicol %}}

---

## Modellazione dei side effect tramite monadi

Il design pattern delle monadi consiste nel trasformare il programma in una struttura dati immutabile che __descrive__ i side effect che possono verificarsi:

- Un programma diventa __un'entità di prima classe__
- I programmi possono essere descritti e composti in maniera __modulare__
- È possibile __modificare la semantica__ di un programma interpretandolo diversamente

---

Una funzione realizzata seguendo questo approccio non "mente" sul proprio comportamento: specifica in maniera astratta il __contesto computazionale__ all'interno del quale può aver luogo e quali side effect possono verificarsi

```scala
// f : Double => Double
def f(x: Double): Double =
  appendToFile("path/to/file", "Effect!")
  Math.cos(x)
```

```scala
// f : Double => M[Double]
def f[M[_]: FileSystem: Console: Monad](x: Double): M[Double] = for
    _ <- FileSystem.appendToFile("path/to/file", "Effect!")
  yield Math.cos(x)
```

---

<section data-noprocess data-auto-animate>
  <h2>Effect System</h2>
  <p>L'approccio monadico permette di imporre un effect system </p>
  <ul>
    <li>Permette di individuare esattamente quali side effect può avere una funzione</li>
    <li>Permette di capire quali funzioni sono pure e quali no</li>
    <li>Impedisce di mescolare inavvertitamente codice puro e con side effect</li>
  </ul>

  <div class="outer-circle">
    Imperative shell
    <div class="inner-circle">Functional core</div>
  </div>
</section>

---

<!--

{{< slide transition="none" >}}

```scala
def computation() =
  val managed = Resource.managed(open(fileUrl))(close(_))
  managed.use { data =>
    breadthFirstSearch(data)
  }
```

---

{{< slide transition="none" >}}

```scala
def computation() =
  ZIO.foreach(fileUrls) { fileUrl => 
  //  ▔▔▔▔▔▔▔ Per riapplicare la stessa logica a tutti gli URL 
    val managed = Resource.managed(open(fileUrl))(close(_))
    managed.use { data =>
      breadthFirstSearch(data)
    }
  }
```

---

{{< slide transition="none" >}}

```scala
def computation() =
  ZIO.foreachPar(fileUrls) { fileUrl => 
  //  ▔▔▔▔▔▔▔▔▔▔ Per gestire tutti gli URL in parallelo
    val managed = Resource.managed(open(fileUrl))(close(_))
    managed.use { data =>
      breadthFirstSearch(data)
    }
  }
```

---

{{< slide transition="none" >}}

```scala
def computation() =
  ZIO.foreachParN(fileUrls)(20) { fileUrl => 
  //  ▔▔▔▔▔▔▔▔▔▔▔ Per limitare il parallelismo
    val managed = Resource.managed(open(fileUrl))(close(_))
    managed.use { data =>
      breadthFirstSearch(data)
    }
  }
```

---

{{< slide transition="none" >}}

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachParN(20)(fileUrls) { fileUrl => 
    val managed = Resource.managed(open(fileUrl))(close(_))
    val robust = managed.retry(policy)
    //           ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ Retry nell'acquisizione della risorsa
    robust.use { data =>
      breadthFirstSearch(data)
    }
  }
```

---

{{< slide transition="none" >}}

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachParN(20)(fileUrls) { fileUrl => 
    val managed = Resource.managed(open(fileUrl))(close(_))
    val robust = managed.retry(policy).timeout(30.seconds)
    //                                 ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ Limite al tempo massimo
    robust.use { data =>
      breadthFirstSearch(data)
    }
  }
```

---

{{< slide transition="none" >}}

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachParN(20)(fileUrls) { fileUrl => 
    val managed = Resource.managed(open(fileUrl))(close(_))
    val robust = managed.retry(policy).timeout(30.seconds)
    robust.use { data =>
      breadthFirstSearch(data).race(depthFirstSearch(data))
      //                       ▔▔▔▔ Concorrenza fra due azioni
    }
  }
```

---

{{< slide transition="none" >}}

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachParN(20)(fileUrls) { fileUrl => 
    val managed = Resource.managed(open(fileUrl))(close(_))
    val robust = managed.retry(policy).timeout(30.seconds)
    robust.use { data =>
      breadthFirstSearch(data).race(depthFirstSearch(data))
    }
  }.timeout(10.minutes)
  //▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ Timeout all'intero processo
```
-->