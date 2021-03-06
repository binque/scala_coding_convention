**语法特性**

1） 定义隐式类时，应该将构造函数的参数声明为val。

2）使用for表达式；如果需要条件表达式，应将条件表达式写到for comprehension中：
```scala
//not good
for (file <- files) {
     if (hasSoundFileExtension(file) && !soundFileIsLong(file)) {
        soundFiles += file
     }
}

//better
for {
     file <- files
     if hasSoundFileExtension(file)
     if !soundFileIsLong(file)
} yield file
```

通常情况下，我们应优先考虑filter, map, flatMap等操作，而非for comprehension：
```scala
//best
files.filter(hasSourceFileExtension).filterNot(soundFileIsLong) 
```

3) 避免使用isInstanceOf，而是使用模式匹配，尤其是在处理比较复杂的类型判断时，使用模式匹配的可读性更好。   
```scala
//avoid
if (x.isInstanceOf[Foo]) { do something …

//suggest
def isPerson(x: Any): Boolean = x match {
  case p: Person => true
  case _ => false
} 
```

4）以下情况使用abstract class，而不是trait：
* 想要创建一个需要构造函数参数的基类
* 代码可能会被Java代码调用

5) 如果希望trait只能被某个类（及其子类）extend，应该使用self type：
```scala
trait MyTrait {
    this: BaseType =>
}
```

如果希望对扩展trait的类做更多限制，可以在self type后增加更多对trait的混入：
```scala
trait WarpCore {
     this: Starship with WarpCoreEjector with FireExtinguisher =>
}

// this works
class Enterprise extends Starship
    with WarpCore
    with WarpCoreEjector
    with FireExtinguisher

// won't compile
class Enterprise extends Starship
     with WarpCore
     with WarpCoreEjector
```

如果要限制扩展trait的类必须定义相关的方法，可以在self type中定义方法，这称之为structural type（类似动态语言的鸭子类型）:
```scala
trait WarpCore {
     this: {
        def ejectWarpCore(password: String): Boolean
        def startWarpCore: Unit
     } =>
}

class Starship
class Enterprise extends Starship with WarpCore {
     def ejectWarpCore(password: String): Boolean = {
          if (password == "password") { println("core ejected"); true } else false }
     def startWarpCore { println("core started") }
}
```

6) 对于较长的类型名称，在特定上下文中，以不影响阅读性和表达设计意图为前提，建议使用类型别名，它可以帮助程序变得更简短。例如：
```scala
class ConcurrentPool[K, V] {
   type Queue = ConcurrentLinkedQueue[V]
   type Map   = ConcurrentHashMap[K, Queue]  
}
```

7) 如果要使用隐式参数，应尽量使用自定义类型作为隐式参数的类型，而避免过于宽泛的类型，如String，Int，Boolean等。
```scala
//suggestion
def maxOfList[T](elements: List[T])
         (implicit orderer: T => Ordered[T]): T =
   elements match {
      case List() =>
         throw new IllegalArgumentException("empty list!")
      case List(x) => x
      case x :: rest =>
         val maxRest = maxListImpParm(rest)(orderer)
         if (orderer(x) > maxRest) x
         else maxRest
   }

//avoid
def maxOfListPoorStyle[T](elements: List[T])
        (implicit orderer: (T, T) => Boolean): T
```

8) 对于异常的处理，Scala除了提供Java风格的try...catch...finally之外，还提供了allCatch.opt、Try…Success…Failure以及Either…Right…Left等风格的处理方式。其中，Try是2.10提供的语法。根据不同的场景选择不同风格：

* 优先选择Try风格。Try很好地支持模式匹配，它兼具Option与Either的特点，因而既提供了集合的语义，又支持模式匹配，又提供了getOrElse()方法。同时，它还可以组合多个Try，并支持运用for combination。例如：
``` scala
val z = for {
    a <- Try(x.toInt) 
    b <- Try(y.toInt)
} yield a * b
val answer = z.getOrElse(0) * 2
```

* 如果希望清楚的表现非此即彼的特性，应考虑使用Either。注意，约定成俗下，我们习惯将正确的结果放在Either的右边（Right既表示右边，又表示正确）

* 如果希望将异常情况处理为None，则应考虑使用allCatch.opt。例如：
```scala
import scala.util.control.Exception._

def readTextFile(f: String): Option[List[String]] =     
    allCatch.opt(Source.fromFile(f).getLines.toList)
```

* 如果希望在执行后释放资源，从而需要使用finally时，考虑try…catch...finally，或者结合try...catch...finally与Either。例如：
```scala
  private def executeQuery(conn: Connection, sql: String): Either[SQLException, ResultSet] = {
    var stmt: Statement = null
    var rs: ResultSet = null
    try {
      stmt = conn.createStatement()
      rs = stmt.executeQuery(sql)
      Right(rs)
    } catch {
      case e: SQLException => {
        e.printStackTrace()
        Left(e)
      }
    } finally {
      try {
        if (rs != null) rs.close()
        if (stmt != null) stmt.close()
      } catch {
        case e: SQLException => e.printStackTrace()
      }
    }
  }
```

为避免重复，还应考虑引入Load Pattern。 