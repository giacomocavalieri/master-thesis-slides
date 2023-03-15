+++
weight = 1
+++

## Outline della tesi

- Introduzione graduale e tramite esempi del concetto di _monade_ e _stack di monadi_
- Analisi dell'approccio _Monad Transformer Library (MTL)_ come meccanismo per tracciare gli effetti
- Analisi dell'approccio delle _free monad_ come meccanismo per tracciare gli effetti
- Introduzione dell'approccio degli _effetti algebrici_ e vantaggi rispetto all'approccio monadico

---

## Side effect nei linguaggi di programmazione

> If a program has no __side effects__ there’s no point in running it.
> You have a black box, you press go and it gets hot but there’s no output!
> _Simon Peyton-Jones_

```scala
def appendToFile(file: String, line: String): Unit =
  log(LogLevel.Info, f"Appending $line to $file")
  Using(FileWriter(file, true))(_.write(f"$line\n"))
```

---

> __Side effects are lies__. Your function promises to do one thing, but it also does other hidden things.
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

## Obiettivi della tesi

- Fornire una visione d'insieme sulle principali tecniche ad oggi adottate per modellare in maniera esplicita i side effect delle funzioni: _Monad Transformer Library (MTL)_ e _free monad_
- Analizzare il nuovo approccio emergente degli _effetti algebrici_
- Mostrare l'efficacia di questi approcci come strumenti di design in un paradigma _"orientato agli effetti"_

---

## Modellazione esplicita dei side effect tramite monadi

L'idea alla base di questi approcci consiste nel trasformare il programma in una struttura dati immutabile che _descrive_ i side effect che possono verificarsi:

- Un programma diventa _un'entità di prima classe_
- I programmi possono essere descritti e composti in maniera _modulare_
- È possibile _modificare la semantica_ di un programma interpretandolo diversamente

---

## MTL e Free Monad

Una funzione realizzata seguendo questo approccio non "mente" sul proprio comportamento: specifica in maniera astratta i side effect che possono aver luogo

```scala
def appendToFile[Effs[_]: FileSystem: Logging](file: String, line: String)
  //            ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ I possibili side effect
    : Program[Effs, Unit] =
  //  ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ Restituisce la descrizione di un programma
  for
    _ <- Logging.log(LogLevel.Info, f"Appending $line to $file")
    _ <- FileSystem.use(file)(_.write(f"$line\n"))
  yield ()
```

---

Vengono disaccoppiati la _descrizione_ del programma e la sua _interpretazione_: la descrizione specifica in maniera astratta _quali_ effetti devono aver luogo e in un secondo momento è possibile stabilie _come_ dovranno essere interpretati

```scala
def interpret() = 
  program.nextInstruction match
    Return(result) => result
    SideEffect(effect, continuation) => effect match
      Log(logLevel, message) => 
        println(f"[$logLevel] $message")
        continuation(()).interpret()
      OpenFile(file, mode)   => ...
      CloseFile(fileHandle)  => ...
```

---

## Effetti algebrici

L'approccio basato su effetti algebrici ha l'obiettivo di _tracciare esplicitamente e in maniera automatica_ gli effetti, senza dover ricorrere all'utilizzo di monadi

```kotlin
// appendToFile : (string, string) -> <logging, fileSystem> ()
fun appendToFile(file, line)
  log(LogLevel.Info, "Appending $line to $file")
  using(file) { fn(handle) handle.write(content) }
```

---


Gli approcci basati su effetti algebrici presentano diversi vantaggi rispetto alle controparti monadiche

- Il codice _appare imperativo_ e non richiede complesse annotazioni di tipo
- _Inferenza automatica_ dei tipi (ed effetti!)
- _Minor carico cognitivo_ per il programmatore

---

```scala
// appendToFile : (String, String) => Unit
def appendToFile(file: String, line: String): Unit =
  log(LogLevel.Info, f"Appending $line to $file")
  Using(FileWriter(file, true))(_.write(f"$line\n"))
```

{{% fragment %}}
```scala
// appendToFile : (String, String) => Program[Effs, Unit]
def appendToFile[Effs[_]: FileSystem: Logging](file: String, line: String)
    : Program[Effs, Unit] =                    
  for
    _ <- Logging.log(LogLevel.Info, f"Appending $line to $file")
    _ <- FileSystem.use(file)(_.write(f"$line\n"))
  yield ()
```
{{% /fragment %}}

{{% fragment %}}
```kotlin
// appendToFile : (string, string) -> <logging, fileSystem> ()
fun appendToFile(file, line)
  log(LogLevel.Info, "Appending $line to $file")
  using(file) { fn(handle) handle.write(content) }
```
{{% /fragment %}}

---

<!--
---

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di definire DSL per descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare_.

```scala
def computation() =
  val managed = Resource.managed(open(fileUrl))(close(_))
  managed.use { data =>
    breadthFirstSearch(data)
  }
```

---

{{< slide transition="none" >}}

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachPar(fileUrls) { fileUrl => 
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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachPar(fileUrls) { fileUrl => 
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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachPar(fileUrls) { fileUrl => 
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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

```scala
val policy = Schedule.recurs(10) && Schedule.exponential(100.millis)

def computation() =
  ZIO.foreachPar(fileUrls) { fileUrl => 
    val managed = Resource.managed(open(fileUrl))(close(_))
    val robust = managed.retry(policy).timeout(30.seconds)
    robust.use { data =>
      breadthFirstSearch(data).race(depthFirstSearch(data))
    }
  }.timeout(5.minutes)
  //▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔ Timeout all'intero processo
```
-->

---

## Adozione

- L'approccio monadico rappresenta lo _standard de facto_ in linguaggi funzionali puri come Haskell
- In Scala sono state sviluppate _librerie che implementano l'approccio monadico_: ZIO, Cats Effect, etc.
- L'approccio degli effetti algebrici sta ricevendo crescente attenzione

> The WasmFX project extends WebAssembly with effect handlers as a unifying mechanism to enable efficient compilation of control idioms, such as __async/await, generators/iterators, first-class continuations__, etc.
> _WasmFX Overview_

---

> We’ve managed to achieve __higher decoupling__ and independence for our various services and dev teams that have very different use-cases while maintaining a single uniform infrastructure in place. Our Kafka infrastructure is called Greyhound and was recently __completely re-written using ZIO__.
> _Natan Silnitsky - Backend Infra Developer, Wix_

> Free Monads are fast, reliable, simple and expressive. [...] This technology __helped the company to grow up__ [...] and helped us to easily reason about the domain, the logic, the business goals and requirements.
> _Alexander Granin - Senior Software Architect, Juspay_

---

## Conclusioni

- Tracciare i side effect delle funzioni ne _rende esplicito il comportamento_, semplificando la possibilità di ragionare sulle loro proprietà e rifattorizzare il codice
- Gli effetti possono essere sfruttati come _strumento di design_ e organizzazione del software in un paradigma "orientato agli effetti"
- L'approccio basato su effetti algebrici rappresenta una promettente _sintesi fra la semplicità di scrittura_ del codice imperativo _e i vantaggi della gestione esplicita degli effetti_
