+++
weight = 1
+++

# Cool equations

Displayed equations are wrapped in double-\$

$$\frac{n!}{k!(n-k)!} = \binom{n}{k}$$  

Inline equations like $E=mc^2$ are wrapped in single-\$

---

# Code example

```kotlin
interface Foo

fun printHelloWorld(): Unit {
    val variable: String = "Hello World"
    println(variable)
}
```

Scala example

```scala
trait Monad[F[_]]

def main(args: Array[String]) = {
    println("Hello World")
}
```

---

# Mermaid

{{< mermaid >}}
graph TD
    A[Client] --> B[Load Balancer]
    B --> C[Server01]
    B --> D[Server02]
{{< /mermaid >}}

---

# Style

This is a **bold** text.

This is an *italic* text.

This is a ~~strikethrough~~ text.

This is a [link](https://www.google.com).

---

# Multi column

{{< multicol >}}
{{% col %}}
Column 1
{{% /col %}}

{{< col >}}
Column 2
{{< /col >}}
{{< /multicol >}}
