# Kotlin相关基础

### 在 Kotlin 中 val 和 var 有什么区别？
`val` 用于声明只读变量，不能被重新赋值，而 var 用于声明可变变量，可以被重新赋值。

    val myString = "Hello World"
    myString = "Hello Kotlin" // 编译会报错

    var myString = "Hello World"
    myString = "Hello Kotlin" // 正常运行

### 在 Kotlin 中 Lateinit 和 lazy 有什么区别？
`lateinit` 用于在构造函数之外初始化一个不可为空的属性，`lazy` 用于创建一个属性，其值仅在首次访问时才会使用。

    class MyClass {
        lateinit var myProperty: String
        fun initialize() {
            myProperty = "Hello"
        }
    }
    val myClass = MyClass()
    myClass.initialize()
    println(myClass.myProperty) // Output: Hello

    val myLazyProperty: String by lazy {
        "Hello"
    }
    println(myLazyProperty) // Output: Hello

### 说一下 Kotlin 中的操作符 ?.
操作符 `?.` 用于安全地访问对象的属性或方法。它在访问对象之前检查对象是否为空，如果是，则返回 `null`。

### 说一下 Android 中 Coroutines
`Coroutines` 是一个轻量级的并发框架，它允许你以更可读性和可管理性的方式编写异步代码。它们用于执行长时间运行的任务，例如网络请求或数据库操作，而不会阻塞主线程。这使应用程序响应性更快，提高了用户体验。

    suspend fun getDataFromApi(): String {
        return withContext(Dispatchers.IO) {
            // perform network request
        }
    }

### Kotlin 中的 Companion Object 和 Object 声明有什么区别？
`Companion Object` 是与一个类相关联的单例对象，而 `Object` 则创建了一个不与任何类相关联的单例对象。`Companion Object` 可以访问其关联类的私有成员，而 `Object` 不能。

    class MyClass {
        companion object {
            fun create(): MyClass {
                // access private members of MyClass
            }
        }
    }

    object MySingleton {
        fun doSomething() {
            // not associated with any class
        }
    }

### 说一下 Kotlin 中密封类
密封的类用于表示给定类型的集合。它们用于以更明确和可读的方式表达变量或对象的可能状态。它们只能有一组有限的子类，并且在与密封类相同的文件中定义。

    sealed class ApiResponse {
        data class Success(val data: Any): ApiResponse()
        data class Error(val error: String): ApiResponse()
        object Loading: ApiResponse()
    }

    fun handleApiResponse(response: ApiResponse) {
        when (response) {
            is ApiResponse.Success -> handleSuccess(response.data)
            is ApiResponse.Error -> handleError(response.error)
            ApiResponse.Loading -> showLoading()
        }
    }

### 普通类和  object class 有什么区别
普通类是用于创建对象的模板，而 `object class` 是在整个应用程序中只能有一个实例的单例类。普通类用 `class` 关键字声明，而 `object class` 用 `Object` 关键字声明。

    class MyRegularClass { // 普通类
        //properties and methods
    }

    object MyObjectClass { // object class
        //properties and methods
    }

### 数据类和普通类有什么区别
数据类是专门为保存数据而设计的类，并自动生成 `equals`、`hashCode` 和 `toString` 等等方法。普通类必须手动实现。

    data class User(val name: String, val age: Int)

    class Employee(val name: String, val age: Int) {
        override fun equals(other: Any?) = ...
        override fun hashCode() = ...
        override fun toString() = ...
    }

### 说一下 Kotlin 中的 when 表达式？
`when` 表达式是在 Kotlin 中表达 `switch` 语句的简洁方式。根据匹配值执行相应的代码分支。它还包括一个 `else` 分支作为一个兜底分支。

    val x = 2
    when (x) {
        1 -> print("x is 1")
        2 -> print("x is 2")
        else -> print("x is neither 1 nor 2")
    }

### 说一下 Kotlin 中扩展函数
Kotlin 中的扩展函数允许给现有类添加新功能，而无需继承或使用装饰器等设计模式。它们是在类之外定义的。

    fun String.addExclamation(): String {
        return "$this!"
    }

    val myString = "Hello"
    println(myString.addExclamation()) //prints "Hello!"

### with 函数在 Kotlin 中如何使用
Kotlin 中的 `with` 函数用于调用对象上的多个方法，而不必为每个方法调用重复对象的名称。以简洁易读的方式对对象执行多个操作。

    val myTextView = TextView(this)
    with(myTextView) {
        text = "Hello"
        textSize = 20f
        setPadding(10, 10, 10, 10)
    }

### 在 Kotlin 中如何使用 Apply 函数？
Kotlin 中的 Apply 函数用于调用对象上的多个方法并返回对象本身。它类似于 with 函数，但它返回被操作的对象。

    val myTextView = TextView(this).apply {
        text = "Hello"
        textSize = 20f
        setPadding(10, 10, 10, 10)
    }

在本例中，使用 `Apply` 函数初始化了 `TextView` 对象，然后返回对象本身 `TextView` 继续使用。

### 在 Kotlin 中 let 函数是如何使用？
Kotlin 中的 `let` 函数仅在对象不为空时对该对象执行操作。它有助于避免为 null 并使代码更具可读性。它接受一个 `lambda` 表达式作为参数，并将对象作为参数传递给 `lambda`。

    val myString: String? = "Hello"
    myString?.let { print(it) } // prints "Hello"
    val myString: String? = null
    myString?.let { print(it) } // does nothing

