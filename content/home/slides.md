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

## Obiettivi della tesi

- Fornire una visione d'insieme sulle principali tecniche ad oggi adottate per modellare in maniera esplicita i side effect delle funzioni: _Monad Transformer Library (MTL)_ e _free monad_
- Analizzare il nuovo approccio emergente degli _effetti algebrici_
- Mostrare l'efficacia di questi approcci come strumenti di design in un paradigma _"orientato agli effetti"_

---

## Modellazione esplicita dei side effect

L'idea alla base di questi approcci consiste nel trasformare il programma in una struttura dati immutabile che _descrive_ i side effect che possono verificarsi:

- Un programma diventa _un'entità di prima classe_
- I programmi possono essere descritti e composti in maniera _modulare_
- È possibile _modificare la semantica_ di un programma interpretandolo diversamente

---

Una funzione realizzata seguendo questo approccio non "mente" sul proprio comportamento: specifica in maniera astratta il _contesto computazionale_ all'interno del quale può aver luogo e quali side effect possono verificarsi

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

## Killer feature: concorrenza strutturata

Questo approccio di modellazione dei side effect permette di definire DSL per descrivere complessi meccanismi di concorrenza in maniera _concisa, dichiarativa e modulare._

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

---

## Conclusioni

- Rendere espliciti i side effect delle funzioni ne _rende esplicito il comportamento,_ semplificando la possibilità di ragionare sulle sue proprietà e rifattorizzare il codice
- Come nel caso della concorrenza strutturata la gestione esplicita degli effetti permette di _definire dei DSL ad hoc per il dominio affrontato_
- Gli effetti possono quindi essere adottati come _strumento di design_ e organizzazione del software in un paradigma "orientato agli effetti"
