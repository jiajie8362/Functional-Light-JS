# Functional-Light JavaScript
# Chapter 5: Reducing Side Effects

在第二章中，我们讨论了一个函数如何能够拥有 `return` 值之外的输出。至此你应当对一个函数的 FP 定义感到非常舒适了，那么这种副输出 —— 副作用！—— 的想法应当发出臭味了。

我们将要检视各种不同形式的副作用，并看看为什么它们对我们代码的质量和可读性有害。

但别让我在这里喧宾夺主。这一章的要点是：写出一个没有副作用的程序是不可能的。好吧，不是不可能；你当然能。但是那样的程序将不会有什么用，也无法观察。如果你写了一个副作用为零的程序，那么你将无法说出它与一个被删除的或空的程序有什么区别。

FP 程序员不会消灭所有的副作用。他们的目标是尽量地限制它们。为此，我们需要完全地理解它们。

## Effects On The Side, Please

因与果：我们人类对周围世界可以做出的最基础的，直觉的观察之一。将一本书推离桌子的边缘，它将掉到地上。你不需要物理学学位就能知道，其原因是你推了这本书而且重力的效果将它拉到了地上。这里有一个清晰且直接的关系。

在程序中，我们也完全是在处理因与果。如果你调用一个函数（因），它就会在屏幕上打印一个消息（果）。

在阅读一段程序时，读者能够清晰地定位每一个因与每一个果是极其重要的。在通读程序之后不能轻易地看出因果之间的直接联系 —— 任意程度的这种事情都会使你程序的可读性降低。

考虑如下代码：

```js
function foo(x) {
	return x * 2;
}

var y = foo( 3 );
```

在这个不起眼的程序中，以下事情是可以立即明确的：使用值 `3` 调用 foo（原因）将会有返回值 `6` 的效果，然后它被赋值给 `y`（结果）。这里没有任何歧义。

但现在：

```js
function foo(x) {
	y = x * 2;
}

var y;

foo( 3 );
```

这段程序拥有完全一样的结果。但这里有一个巨大的不同。原因与结果脱节了。结果是间接的。这种设置 `y` 的方式就是我们称之为副作用的东西。

**注意：** 当一个函数引用一个它外部的变量时，它称为一个自由变量。不是所有自由变量都是坏的，但我们将非常小心地对待它们。

要是我给你一个你看不到代码的函数 `bar(..)` 的调用引用，但我告诉你它没有这样的间接副作用，而只有一个明确的 `return` 值效果呢？

```js
bar( 4 );			// 42
```

因为你知道 `bar(..)` 的内部不会制造任何副作用，所以你现在可以用更加直接了当的方式推理任何一个像 `bar(..)` 这样的调用。但如果你不知道 `bar(..)` 没有副作用，那么要理解调用它的结果，你就不得不去阅读并剖析它所有的逻辑。对读者来说这是额外的思维负担。

**一个带有副作用的函数的可读性要差一些**，因为它要求更大的阅读量才能理解程序。

但是问题会变得更严重。考虑如下代码：

```js
var x = 1;

foo();

console.log( x );

bar();

console.log( x );

baz();

console.log( x );
```

你对每一个 `console.log(x)` 将要打印出的值有多确定？

正确答案是：完全无法确定。如果你不能确定 `foo()`、`bar()`、和 `baz()` 是否有副作用，你就无法保证 `x` 在每一步中是什么。除非你检查每一个函数的实现，**并且** 从第一行开始追踪程序，一边走一边监视状态的所有改变。

换言之，最终的 `console.log(x)` 是不可能被分析和预测的，除非你在大脑中将整个程序执行到那个地方。

猜猜谁更擅长运行你的程序？JS 引擎。再猜猜谁不擅长运行你的程序？你的代码的读者。而且，你选择在这些函数中的一个或几个里面编写带有（潜在）副作用的代码，意味着你使读者背上了这样一种负担：他们为了理解某一行，就不得不在大脑中将你的程序完整第运行到那一行。

如果 `foo()`、`bar()`、和 `baz()` 都是无副作用的，它们不会影响 `x`，这意味着我们不必在大脑中执行它们来跟踪 `x` 上发生了什么。这样思维成本更低，而且是得代码更具可读性。

### Hidden Causes

输出、状态的改变，是最常被提到的副作用的表现。但是另一种有损可读性的做法是一些人称之为侧因（side causes）的东西。考虑如下代码：

```js
function foo(x) {
	return x + y;
}

var y = 3;

foo( 1 );			// 4
```

`y` 没有被 `foo(..)` 改变，所以这不是我们以前看到的那种副作用。但现在，`foo(..)` 的调用实际上依赖于 `y` 的存在和当前状态。如果稍后我们这么做：

```js
y = 5;

// ..

foo( 1 );			// 6
```

也许我们会被 `foo(1)` 在调用与调用之间返回不同的结果吓到？

`foo(..)` 有一个损害可读性的间接起因。如果不仔细检查 `foo(..)` 的实现，读者就看不到是什么原因在影响着输出的结果。*看起来* 参数 `1` 是唯一的起因，但事实证明它不是。

为了增强可读性，所有将会影响 `foo(..)` 输出结果判定的起因都应当作为 `foo(..)` 的直接的、明显的输入。代码的读者将可以清楚地看到起因和结果。

#### Fixed State

避免侧因意味着函数 `foo(..)` 不能引用任何自由变量吗？

考虑这段代码：

```js
function foo(x) {
	return x + bar( x );
}

function bar(x) {
	return x * 2;
}

foo( 3 );			// 9
```

很清楚，对于 `foo(..)` 和 `bar(..)` 两者来说唯一的直接起因就是形式参数 `x`。那么 `bar(x)` 的调用呢？`bar` 只是一个标识符，而且在 JS 中它甚至默认不是一个常量（不可再赋值的变量）。函数 `foo(..)` 依赖于 `bar` 的值 —— 一个引用第二个函数的变量 —— 一个自由变量。

那么这个程序是依赖于侧因的吗？

我说不。即使使用其他函数来覆盖变量 `bar` 的值是 *可能* 的，我也没在这段代码中这么做，这不是我的常见做法，也没有这样的先例。对于我所有的意图和目的来说，我的函数就是常量（从不重新赋值）。

考虑如下代码：

```js
const PI = 3.141592;

function foo(x) {
	return x * PI;
}

foo( 3 );			// 9.424776000000001
```

**注意：** JavaScript 有一个 `Math.PI` 內建值，我们在这本书里使用 `PI` 的例子只是为了方便展示。在实际应用中，总是使用 `Math.PI` 而不是定义你自己的！

这个代码段呢？`PI` 是 `foo(..)` 的一个侧因吗？

两个观点将帮助我们以一种合理的方式回答这个问题：

1. 考虑你可能发起的每一个 `foo(3)` 调用。它们将总是返回值 `9.424..` 吗？**是的。** 每一次。如果你给它相同的输入（`x`），它就总是返回相同的输出。

2. 你能使用 `PI` 的立即值替换每一个用到 `PI` 的地方，而且程序还能 **完全** 和以前一样运行吗？**是的。** 这个程序没有其他部分可以改变 `PI` 的值 —— 确实，因为它是一个 `const`，不能被重新赋值 —— 所以这里的变量 `PI` 只是为了可读性/可维护性存在的。它的只可以被内联而不改变程序的任何行为。

我的结论：这里的 `PI` 没有违反最小化/避免副作用（或侧因）的精神。前一个代码段中的 `bar(x)` 也没有。

在这两种情况下，`PI` 和 `bar` 都不是程序状态的一部分。它们是固定的，不可重新赋值的引用（“常量”）。如果它们贯穿程序始终都不改变，我们就不必费心将它们视为可变状态追踪。因此，它们没有损害我们的可读性。而且它们不可能是由于变量以意外的方式改变而引起的 bug 的源头。

**注意：** 依我看，上面 `const` 的使用并不是 `PI` 没有成为侧因的理由；`var PI` 也会得出相同的结论。没有给 `PI` 重新赋值才是重要的，而不是没有这种能力。我们将会在后面的章节中讨论 `const`。

#### Randomness

你可能从没考虑过，但随机性是不纯的。一个使用了 `Math.random()` 的函数绝不可能是纯函数，因为你不能基于它的输入保证/预测它的输出。所以任何生成唯一随机 ID 等东西的代码，根据定义都将被认为依赖于你程序的侧因。

在计算机科学中，我们使用称为伪随机算法的东西来生成随机数。事实证明随机性相当难以实现，所以我们只是使用一些产生看起来随机的值的复杂算法来假冒它。这些算法计算出一些很长的数字流，但其中的秘密是，如果你知道它的起点，这些序列实际上是可以预测的。这个起点称为种子。

有些语言允许你为随机数的生成指定种子值。如果你总是指定相同的种子，那么你将总是从后续的“随机数”生成中得到相同的输出序列。这对测试来说具有不可估量的价值，但是对现实世界中程序的使用有不可估量的危险。

在 JS 中，`Math.random(..)` 计算出的随机性是基于一个间接输入的，因为你不能指定种子。因此，我们不得不将內建的随机数生成视为一种不纯的侧因。

### I/O Effects

可能还不是很明显，但是副作用/侧因的最常见形式是 I/O（输入/输出）。一个没有 I/O 的程序是完全无意义的，因为它完成的工作无论以什么方式都不可见。有用的程序必须至少拥有输出，而且可能还需要输入。输入以一种侧因，而输出是一种副作用。

在浏览器的 JS 程序中最常见的输入就是用户事件（鼠标，键盘），而输出就是 DOM。如果你用 Node.js 比较多，那么你更可能从文件系统、网络连接、和/或 `stdin`/`stdout` 流中接收输入与发送输出。

事实上，这些源头既可以是输入也可以是输出，同为因果。例如 DOM。我们更新（副作用）一个 DOM 元素来向用户展示一段文字或一张图片，但 DOM 的当前状态对于这些操作来说也是一种隐含的输入（侧因）。

### Side Bugs

侧因与副作用导致 bug 的场景会因它们在程序中存在的形态而不同。但让我们来检视一种场景来展示一下这些灾难，希望它们能帮你在你自己的程序中找出相似的错误。

考虑如下代码：

```js
var users = {};
var userOrders = {};

function fetchUserData(userId) {
	ajax( "http://some.api/user/" + userId, function onUserData(userData){
		users[userId] = userData;
	} );
}

function fetchOrders(userId) {
	ajax( "http://some.api/orders/" + userId, function onOrders(orders){
		for (let i = 0; i < orders.length; i++) {
			// keep a reference to latest order for each user
			users[userId].latestOrder = orders[i];
			userOrders[orders[i].orderId] = orders[i];
		}
	} );
}

function deleteOrder(orderId) {
	var user = users[ userOrders[orderId].userId ];
	var isLatestOrder = (userOrders[orderId] == user.latestOrder);

	// deleting the latest order for a user?
	if (isLatestOrder) {
		hideLatestOrderDisplay();
	}

	ajax( "http://some.api/delete/order/" + orderId, function onDelete(success){
		if (success) {
			// deleted the latest order for a user?
			if (isLatestOrder) {
				user.latestOrder = null;
			}

			userOrders[orderId] = null;
		}
		else if (isLatestOrder) {
			showLatestOrderDisplay();
		}
	} );
}
```

我打赌对一些读者来说这里的潜在的 bug 之一是相当明显的。如果回调 `onOrders(..)` 在回调 `onUserData(..)` 之前运行，它就会试图将一个 `latestOrder` 属性添加到一个还没有被设置的值上（`user[userId]` 上的 `userData` 对象）。

所以在依赖于侧因/副作用的逻辑中可能发生的一种 bug 形式是两个不同操作（异步或者同步！）的竞合状态，我们期望它们以一种特定的顺序运行，但在某些情况下它们可能以一种不同的顺序运行。有一些策略可以保证操作的顺序，但是在这种场景下顺序的重要性是相当明显的。

这里还有另一个微妙的 bug 可能会咬到我们。你发现了吗？

考虑一下这种调用顺序：

```js
fetchUserData( 123 );
onUserData(..);
fetchOrders( 123 );
onOrders(..);

// later

fetchOrders( 123 );
deleteOrder( 456 );
onOrders(..);
onDelete(..);
```

你看到 `fetchOrders(..)` / `onOrders(..)` 与 `deleteOrder(..)` / `onDelete(..)` 之间的穿插了吗？在我们状态管理的侧因/副作用中，这种潜在的序列暴露出了一个奇怪状态。

在我们设置 `isLatestOrder` 标志，和我们使用它来决定我们是否应当清空 `user` 中用户数据的 `latestOrder` 属性之间存在一个时间的延迟（因为回调）。在这个延迟期间，如果 `onOrders(..)` 被触发，它就可能潜在地改变用户的 `latestOrder` 引用的订单值。而之后在 `onDelete(..)` 被触发时，它将假定它依然需要解除 `latestOrder` 引用。

bug 就是：现在数据（状态）*可能* 已经不同步了。在 `latestOrder` 本应潜在地保持指向来自 `onOrders(..)` 的更新的订单时，这种指向被解除了。

这种 bug 最可怕的地方就是它不会像其他 bug 那样，给你一个程序崩溃的异常。我们就这样得到一个不正确的状态；我们应用程序的行为 “平静地” 坏掉了。

在 `fetchUserData(..)` 与 `fetchOrders(..)` 之间顺序的依赖关系相当明显，而且解决起来直截了当。但 `fetchOrders(..)` 与 `deleteOrder(..)` 之间存在顺序依赖关系可就不那么明显了。它们俩看起来更加独立。而且维护它们的顺序更加棘手，因为你不会提前知道（在 `fetchOrders(..)` 的结果之前）是否应当强制这个顺序。

是的，你可以在 `deleteOrder(..)` 被触发时重新计算 `isLatestOrder` 标志。但是现在你又有了一个不同的问题：你的 UI 状态可能不同步了。

如果你之前已经调用了 `hideLatestOrderDisplay()`，那么现在你需要调用 `showLatestOrderDisplay()` 了，但是仅在新的 `latestOrder` 被实际设定了的情况下调用。所以你现在至少需要跟踪三个状态：被删除的订单是否本来就是“最新”的？“最新”的订单是否被设置了？这两个订单是否不同？当然，这些问题可以解决。但从任何意义上讲它们都不是显而易见的。

所有这些麻烦都是因为我们决定在一组共享的状态上带着侧因/副作用来构建我们的代码而引起的。

函数式程序员痛恨这种侧因/副作用 bug，因为它极大地伤害了我们的可读性，可推理性，可验证性，而最终伤害到了代码的 **可信任性**。这就是为什么它们如此严肃地对待避免侧因/副作用的原则。

有多种不同的策略可以避免/修复侧因/副作用。我们会在本章稍后谈到一些，另外一些在后续章节讨论。我可以确信一件事情：**带着侧因/副作用编写程序经常是我们一般的默认状态**，所以避免它们就要求小心和有意识的努力。

## Once Is Enough, Thanks

如果你必须制造副作用来改变状态，有一类称为幂等性的操作对于限制潜在的麻烦十分有用。如果你对一个值的更新是幂等的，那么数据就能承受来自不同副作用源头的多次同种类的更新。

幂等性的定义有些令人糊涂；与程序员经常使用的含义相比，数学家们使用的含义稍有不同。但是对于函数式程序员来说两种角度都有用。

首先，让我们给出一个计数器的例子，它既不是数学上幂等的也不是程序上幂等的：

```js
function updateCounter(obj) {
	if (obj.count < 10) {
		obj.count++;
		return true;
	}

	return false;
}
```

这个函数通过引用递增 `obj.count` 来改变一个对象，所以它在这个对象上产生了一个副作用。如果 `updateCounter(o)` 被调用了多次 —— 在 `o.count` 小于 `10` 的时候 —— 那么程序的状态每次都会改变。另外，`updateCounter(..)` 的输出是一个布尔值，它不适合作为后续 `updateCounter(..)` 调用的输入。

### Mathematic Idempotence

从数学的视角来看，幂等性意味着一个操作的输出在第一次调用之后就不会再改变了，即使你将这个输出一次又一次地送回这个操作。换言之，`foo(x)` 产生的输出将与 `foo(foo(x))`、`foo(foo(foo(x)))` 等相同。

一个典型的数学的例子是 `Math.abs(..)`（绝对值）。`Math.abs(-2)` 是 `2`，它的结果与 `Math.abs(Math.abs(Math.abs(Math.abs(-2))))` 相同。像 `Math.min(..)`、`Math.max(..)`、`Math.round(..)`、`Math.floor(..)` 和 `Math.ceil(..)` 这样的工具也都是幂等的。

我们可以用与此相同的性质定义一些自己的数学操作：

```js
function toPower0(x) {
	return Math.pow( x, 0 );
}

function snapUp3(x) {
	return x - (x % 3) + (x % 3 > 0 && 3);
}

toPower0( 3 ) == toPower0( toPower0( 3 ) );			// true

snapUp3( 3.14 ) == snapUp3( snapUp3( 3.14 ) );		// true
```

数学上的幂等性 **不** 局限于数学操作。我们可以展示这种形式的幂等性的另一个地方是 JavaScript 的基本类型强制转换：

```js
var x = 42, y = "hello";

String( x ) === String( String( x ) );				// true

Boolean( y ) === Boolean( Boolean( y ) );			// true
```

在本书先前的部分中，我们探索过一个满足这种形式的幂等性的常见的 FP 工具：

```js
identity( 3 ) === identity( identity( 3 ) );	// true
```

一些特定的字符串操作也都是自然幂等的，比如：

```js
function upper(x) {
	return x.toUpperCase();
}

function lower(x) {
	return x.toLowerCase();
}

var str = "Hello World";

upper( str ) == upper( upper( str ) );				// true

lower( str ) == lower( lower( str ) );				// true
```

我们甚至可以用幂等的方式来设计更精巧的字符串格式化操作，比如：

```js
function currency(val) {
	var num = parseFloat(
		String( val ).replace( /[^\d.-]+/g, "" )
	);
	var sign = (num < 0) ? "-" : "";
	return `${sign}$${Math.abs( num ).toFixed( 2 )}`;
}

currency( -3.1 );									// "-$3.10"

currency( -3.1 ) == currency( currency( -3.1 ) );	// true
```

`currency(..)` 展示了一种重要的技术：在某些情况下开发者可以采取额外的步骤来规范化一个输入/输出操作，以保证这个通常不是幂等的操作是幂等的。

无论何处，将副作用限制为幂等操作要比无限制的更新好多了。

### Programming Idempotence

幂等性面向编程的定义是相似的，但没那么正式。与要求 `f(x) === f(f(x))` 不同，这种观点的幂等性只要求 `f(x);` 在程序行为上结果与 `f(x); f(x);` 相同。换言之，在第一次调用 `f(x)` 之后，对 `f(x)` 的后续多次调用不会改变任何东西。

这种角度更符合我们对副作用的观察，以为一个这样的 `f(..)` 操作更像是制造了一个幂等的副作用，而不是必然返回一个幂等的输出值。

这种幂等风格经常被 HTTP 操作（动词）引用，比如 GET 或 PUT。如果一个 HTTP REST API 恰当地按照幂等性的规范指引设计，PUT 被定义为完全替换一个资源的更新操作。这样，一个客户端就可以发送 PUT 请求一次或多次（用相同的数据），而服务器将无论如何都拥有相同的结果状态。

使用编程中更具体的术语考虑这个问题，让我们检视一些副作用操作的幂等性：

```js
// idempotent:
obj.count = 2;
a[a.length - 1] = 42;
person.name = upper( person.name );

// non-idempotent:
obj.count++;
a[a.length] = 42;
person.lastUpdated = Date.now();
```

记住：在这里幂等性的概念是，每个幂等的操作都可以被重复多次，而除了第一次更新以外都不会改变程序的状态。而非幂等操作每次都会改变状态。

那 DOM 的更新呢？

```js
var hist = document.getElementById( "orderHistory" );

// idempotent:
hist.innerHTML = order.historyText;

// non-idempotent:
var update = document.createTextNode( order.latestUpdate );
hist.appendChild( update );
```

这里展示的关键的不同是，幂等更新替换了 DOM 元素的内容。DOM 元素的当前状态无关紧要，因为它被无条件地覆盖了。非幂等操作向元素添加内容；DOM 元素当前的状态隐含地成为了下一个状态的计算的一部分。

以幂等的方式定义你在数据上的操作不总是可能的，但如果你能，它绝对能帮你降低这种可能性 —— 副作用在你最预想不到的时候产生并毁了你的预想。

## Pure Bliss

一个没有侧因/副作用的函数称为一个纯函数。一个纯函数在编程的意义上是幂等的，因为它不能有任何副作用。考虑如下代码：

```js
function add(x,y) {
	return x + y;
}
```

所有的输入（`x` 与 `y`）和输出（`return ..`）都是直接的；没有自由变量的引用。调用 `add(3,4)` 多次与仅调用它一次没有区别。`add(..)` 是纯粹的，而且是编程上幂等的。

然而，不是所有的纯函数都在数学的意义上是幂等的，因为它们不必返回一个适于传递给自己作为输入的值。考虑如下代码：

```js
function calculateAverage(list) {
	var sum = 0;
	for (let i = 0; i < list.length; i++) {
		sum += list[i];
	}
	return sum / list.length;
}

calculateAverage( [1,2,4,7,11,16,22] );			// 9
```

输出 `9` 不是一个数组，所以你不能这样把它传递回去：`calculateAverage(calculateAverage( .. ))`。

正如我们早先讨论过的，一个纯函数 *可以* 引用自由变量，只要那些自由变量不是侧因。

一些例子是：

```js
const PI = 3.141592;

function circleArea(radius) {
	return PI * radius * radius;
}

function cylinderVolume(radius,height) {
	return height * circleArea( radius );
}
```

`circleArea(..)` 引用了自由变量 `PI`，但它是一个常量所以它不是侧因。`cylinderVolume(..)` 引用了自由变量 `circleArea`，它也不是一个侧因，因为这个程序没有这样看待它，而实际上将它作为一个它函数值的常量引用。这两个函数都是纯粹的。

另一个函数引用自由变量但依然纯粹的例子是通过闭包：

```js
function unary(fn) {
	return function onlyOneArg(arg){
		return fn( arg );
	};
}
```

`unary(..)` 自身显然是纯粹的 —— 它唯一的输入是 `fn` 唯一的输出是被 `return` 的函数 —— 但是闭包着自由变量 `fn` 的内部函数 `onlyOneArg(..)` 呢？

它依然是纯粹的，因为 `fn` 绝不会改变。事实上，我们对此有足够的信心，因为从词法上讲这几行是唯一可能对 `fn` 重新赋值的地方。

**注意：** `fn` 是一个函数对象的引用，它默认是可变的。比如程序的其他一些地方 *可能* 会给这个函数对象添加一个属性，从而在技术上“改变”（改变，不是重新赋值）这个值。但是，因为除了能够调用 `fn` 的能力以外，我们不依赖于它任何其他的东西，而且这不可能影响函数的能力，所以对于我们目的来说 `fn` 实际上依然使不变的；它不可能是一个侧因。

另一种准确描述函数纯粹性的常见方式是：**给定相同的输入，它总是产生相同的输出。** 如果你向 `circleArea(..)` 传递 `3`，它将总是输出相同的结果（`28.274328`）。

如果一个函数 *能* 在每次被给予相同输入时产生不同的输出，那么它就不是纯粹的。即便一个函数总是 `return` 相同的值，如果他产生了一个间接的副作用输出，那么程序的状态也会在每次它被调用时改变；这不是纯粹的。

不纯的函数不受欢迎是因为它们使得所有对它们的调用都更难推理。一个纯函数的调用时完全可以预测的。当某人阅读代码看到多个 `circleArea(3)` 调用时，他不必花费任何额外的努力就能搞清楚它的 *每一次* 输出是什么。

### Purely Relative

当我们谈论一个函数是否纯粹的时候必须非常小心。JavaScript 动态值的天性使得隐晦的侧因/副作用太容易发生了。

考虑如下代码：

```js
function rememberNumbers(nums) {
	return function caller(fn){
		return fn( nums );
	};
}

var list = [1,2,3,4,5];

var simpleList = rememberNumbers( list );
```

`simpleList(..)` 看起来是一个纯函数，它是一个内部函数 `caller(..)` 的引用，这个内部函数闭包着自由变量 `nums`。然而，其实有好几种方式可以使 `simpleList(..)` 称为不纯粹的。

首先，我们对纯粹性的断言是基于数组值（同时被 `list` 和 `nums` 引用着）绝不会改变：

```js
function median(nums) {
	return (nums[0] + nums[nums.length - 1]) / 2;
}

simpleList( median );		// 3

// ..

list.push( 6 );

// ..

simpleList( median );		// 3.5
```

当我们改版这个数组时，`simpleList(..)` 调用改变了它的输出。那么，`simpleList(..)` 是纯粹的还是不纯粹的？这要看你的角度。对于给定的一组假设来说它是纯粹的。在任何没有 `list.push(6)` 变化的程序中它都可以使纯粹的。

我们可以通过改变 `rememberNumbers(..)` 的定义来防止这种不纯粹性。一种方式是复制 `nums` 数组：

```js
function rememberNumbers(nums) {
	// make a copy of the array
	nums = nums.slice();

	return function caller(fn){
		return fn( nums );
	};
}
```

但可能潜伏着一个更刁钻的隐藏副作用：

```js
var list = [1,2,3,4,5];

// make `list[0]` be a getter with a side effect
Object.defineProperty(
	list,
	0,
	{
		get: function(){
			console.log( "[0] was accessed!" );
			return 1;
		}
	}
);

var simpleList = rememberNumbers( list );
// [0] was accessed!
```

也许一个更健壮的选项是改变 `rememberNumbers(..)` 的签名，使它不要一上来就接收一个数组，而是接收各个独立的实际参数：

```js
function rememberNumbers(...nums) {
	return function caller(fn){
		return fn( nums );
	};
}

var simpleList = rememberNumbers( ...list );
// [0] was accessed!
```

两个 `...` 的效果是将 `list` 拷贝到 `nums`，而非通过引用传递。

**注意：** 控制台消息的副作用不是来自于 `rememberNumbers(..)` 而是来自于 `...list` 扩散。所以在这种情况下，`rememberNumbers(..)` 和 `simpleList(..)` 都是纯粹的。

但要是改变更难以发现呢？将一个纯函数与一个非纯函数组合 **总是** 产生一个非纯函数。如果我们将一个不纯粹的函数传入本来是纯粹的 `simpleList(..)`，那么它现在就是不纯粹的了：

```js
// yes, a silly contrived example :)
function firstValue(nums) {
	return nums[0];
}

function lastValue(nums) {
	return firstValue( nums.reverse() );
}

simpleList( lastValue );	// 5

list;						// [1,2,3,4,5] -- OK!

simpleList( lastValue );	// 1
```

**注意：** 尽管 `reverse()` 返回了一个反向的数组而看起来安全（就像其他 JS 的数组方法一样），它其实改变了数组而不是创建了一个新的。

我们需要一个更健壮的 `rememberNumbers(..)` 定义来防止 `fn(..)` 通过引用来改变它闭包着的 `nums`：

```js
function rememberNumbers(...nums) {
	return function caller(fn){
		// send in a copy!
		return fn( nums.slice() );
	};
}
```

那么 `simpleList(..)` 纯粹的可靠了！？**没有。** :(

我们只防御了我么可控的副作用（通过引用进行改变）。我们传递的任何函数都可能有另外的副作用会污染 `simpleList(..)` 的纯粹性：

```js
simpleList( function impureIO(nums){
	console.log( nums.length );
} );
```

事实上，没有办法能定义 `rememberNumbers(..)` 而使 `simpleList(..)` 称为成为一个完美的纯函数。

纯粹性就是信心。但在许多情况下我们不得不承认，**我们感到的信心实际上都是相对于我们程序的上下文环境**，以及我们对它知道多少。在（JavaScript 的）实际应用中，函数纯粹性的问题不是关于是否绝对纯粹，而是关于对它纯粹性的信心的范围。

越纯粹越好。你在使一个函数变得纯粹上付出的努力越多，你就在阅读使用它的代码时越有信心，而这将会使这部分代码更具可读性。

## There Or Not

至此，我们将函数纯粹性定义为一个没有侧因/副作用的函数，以及一个只要给出相同输入就总是产生相同输出的函数。这些只是看待一个相同性质的两种不同方式。

但是第三种看待函数纯粹性的方式，而且也许是最广为人接受的定义是，纯函数拥有引用透明性。

引用透明性是指一个函数的调用可以用它的输出值替换，而整个程序的行为不会改变。换句话说，程序的执行发起了对这个函数的调用，还是它的返回值被内联地写在了函数被调用的地方 —— 是不可能知道的。

从引用透明性的视角出发，这两个程序都因为使用了纯函数建造而具有相同的行为

```js
function calculateAverage(list) {
	var sum = 0;
	for (let i = 0; i < list.length; i++) {
		sum += list[i];
	}
	return sum / list.length;
}

var nums = [1,2,4,7,11,16,22];

var avg = calculateAverage( nums );

console.log( "The average is:", avg );		// The average is: 9
```

```js
function calculateAverage(list) {
	var sum = 0;
	for (let i = 0; i < list.length; i++) {
		sum += list[i];
	}
	return sum / list.length;
}

var nums = [1,2,4,7,11,16,22];

var avg = 9;

console.log( "The average is:", avg );		// The average is: 9
```

这两个代码段的唯一区别是，在后者中我们跳过了 `calculateAverage(nums)` 调用而只是内联了它的输出（`9`）。因为程序其余部分的行为完全一样，所以 `calculateAverage(..)` 拥有引用透明性，因此是一个纯函数。

### Mentally Transparent

一个引用透明的纯函数 *可以* 被它的输出替换的概念不意味着它 *就应当被* 替换掉。远远不是。

我们在程序中建造函数而不使用提前计算好的魔法常量，不只是为了对数据的改变作出反应，还是为了恰当抽象的可读性等等。与只是进行明确赋值的那一行比起来，计算那一组数组的平均值的调用使程序的那一部分更具可读性。它给读者讲述了一个故事，`avg` 从何而来，它是什么意思等等。

引用透明性的真正含义是，在你阅读一个程序时，一旦你在思维上计算出了一个纯函数调用的输出是什么，你就不再需要在代码中看到它时考虑这个函数调用究竟在做什么，特别是当它出现许多次的时候。

这个结果变成了某种思维上的 `const` 声明，在阅读的时候你可以透明地将它替换进来，而不必再花思维上的精力计算它。

但愿纯函数这种性质的重要性讲清楚了。我们在试着使我们的程序更易于阅读。我们这么做的一种方式就是让读者少负担一些工作 —— 通过提供一些辅助来跳过不必要的东西，使他们可以将精力集中在重要的东西上。

读者不应该总是重新计算某些不会改变（以及不需要改变）的结果。如果你定义了一个引用透明的纯函数，读者就不必这么做。

### Not So Transparent?

如果一个函数有副作用，但是这种副作用永远不会被观察到，或者程序的其他地方永远不会依赖于这种副作用呢？这个函数依然拥有引用透明性吗？

这里有一个：

```js
function calculateAverage(list) {
	sum = 0;
	for (let i = 0; i < list.length; i++) {
		sum += list[i];
	}
	return sum / list.length;
}

var sum, nums = [1,2,4,7,11,16,22];

var avg = calculateAverage( nums );
```

你发现了吗？

`sum` 是一个 `calculateAverage(..)` 用来完成工作的外部自由变量。但是，每次我们用相同的列表调用 `calculateAverage(..)` 都会得到输出 `9`。而且就程序行为上而言，将 `calculateAverage(nums)` 调用替换为值 `9` 是没有区别的。程序中没有其他任何部分在乎变量 `sum`，所以它是一个不可观测的副作用。

一个不可观测的侧因/副作用像这棵树一样吗？

> 如果一棵树在森林中倒下，但周围没有人听到，那么它发出倒下声音了吗？

根据引用透明性的最狭义的定义，我认为你不得不承认 `calculateAverage(..)` 依然是一个纯函数。但因为我们一直试着使我们的学习不仅学术化，而且要与实用主义平衡，我想这个结论需要更多的观察角度。让我们探索一下。

#### Performance Effects

通常，你会发现这些不可观测的副作用被用来优化一个操作的性能。举例来说：

```js
var cache = [];

function specialNumber(n) {
	// if we've already calculated this special number,
	// skip the work and just return it from the cache
	if (cache[n] !== undefined) {
		return cache[n];
	}

	var x = 1, y = 1;

	for (let i = 1; i <= n; i++) {
		x += i % 2;
		y += i % 3;
	}

	cache[n] = (x * y) / (n + 1);

	return cache[n];
}

specialNumber( 6 );				// 4
specialNumber( 42 );			// 22
specialNumber( 1E6 );			// 500001
specialNumber( 987654321 );		// 493827162
```

这个呆萌的 `specialNumber(..)` 算法是确定性的，而且从对相同的输入总是给出相同的输出这个定义上讲是纯粹的。它从引用透明性的角度上讲也是纯粹的 —— 使用 `22` 替换所有 `specialNumber(42)`，程序的最终结果是相同的。

然而，为了计算某些大一点儿数字这个函数不得不做相当多的工作，特别是 `987654321` 这个输入。如果我们需要在程序中多次取得这个特别的数字，缓存（`cache`）结果可以使后续的调用高效得多。

**注意：** 一个值得深思的有趣的事情：即使对于最纯粹的函数/程序来说，在执行任何给定的操作时 CPU 产生的热量是一种不可避免的副作用吗？那么 CPU 在一个纯粹的操作上花费时间，而使另一个操作发生的延迟呢？

被那么快就假定你可以运行 `specialNumber(987654321)` 计算一次并手动把结果贴在某个变量/常量上。程序通常是高度模块化的，而且全局的可访问作用域通常不是你想要在那些独立的部分之间共享状态的方式。让 `specialNumber(..)` 实现它的自己的缓存（尽管它刚好是使用一个全局变量这么做的！）是这种状态共享的更好的抽象。

重点是如果 `specialNumber(..)` 是程序中唯一可以访问和更新 `cache` 侧因/副作用的部分，那么引用透明性看起来就是成立的，而且这可能看起来可以作为实现纯函数典范的一种可接受的实用的“作弊”手段。

但它应该是吗？

通常，这种性能优化副作用是这样完成的：隐藏结果的缓存使它们在程序的任何其他部分都观察不到。这种处理被称为默记（memoization）。我总是认为这个词是“记忆（memorization）”；我甚至不知道这个词是打哪儿来的，但它确实帮我更好地理解了这个概念。

考虑如下代码：

```js
var specialNumber = (function memoization(){
	var cache = [];

	return function specialNumber(n){
		// if we've already calculated this special number,
		// skip the work and just return it from the cache
		if (cache[n] !== undefined) {
			return cache[n];
		}

		var x = 1, y = 1;

		for (let i = 1; i <= n; i++) {
			x += i % 2;
			y += i % 3;
		}

		cache[n] = (x * y) / (n + 1);

		return cache[n];
	};
})();
```

我们将侧因/副作用 `cache` 包含在了 IIFE `memoization()` 内部，于是现在我们可以确信程序中没有其他部分 *能够* 观察到它了，而不只是它们 *不去* 观察它。

这最后一句话可能听起来在说一件不起眼儿的事，但实际上我认为它可能是 **这整个章节中最重要的观点**。再把它读一遍。

回到这个哲学沉思：

> 如果一棵树在森林中倒下，但周围没有人听到，那么它发出倒下声音了吗？

我在这个类比中得到的启示是：无论有没有发出声音，最好是我们绝不制树倒下而我们不在场的场景；在一棵树倒下时我们将总是听到声响。

减少侧因/副作用的行为本质上不是去制造一个人们无法观察到的程序，而是为了设计一个侧因/副作用尽可能少的程序，因为这会使程序更易于推理。一段带有侧因/副作用的程序 *碰巧* 没有被观察到，对于达成一个 *不能* 观察到它们的程序的目标来说根本没有效果。

如果侧因/副作用可能发生，那么作者与读者就必须在思维上演练它们。如果使它们不可能发生，那么作者与读者就将会对在任何地方什么会发生和什么不会发生有更多的信心。

## Purifying

如果你有一个你无法将之重构为纯函数的非纯函数，你该怎么做？

你需要搞清楚这个函数有什么种类的侧因/副作用。侧因/副作用可能来自于各种途径，词法自由变量、通过引用的修改、或者甚至是 `this` 绑定。我们将看一看解决这些场的方式。

### Containing Effects

如果我们关心的侧因/副作用来自于词法自由变量，而且你可以修改周围的代码，那么你就可以使用作用域来封装它们。

回忆一下：

```js
var users = {};

function fetchUserData(userId) {
	ajax( "http://some.api/user/" + userId, function onUserData(userData){
		users[userId] = userData;
	} );
}
```

纯粹化这段代码的一个选项是在变量和非纯函数周围创建一个包装函数。实质上，这个包装函数必须接收所有它能够操作的“一切”状态。

```js
function safer_fetchUserData(userId,users) {
	// simple, naive ES6+ shallow object copy, could also
	// be done w/ various libs or frameworks
	users = Object.assign( {}, users );

	fetchUserData( userId );

	// return the copied state
	return users;


	// ***********************

	// original untouched impure function:
	function fetchUserData(userId) {
		ajax( "http://some.api/user/" + userId, function onUserData(userData){
			users[userId] = userData;
		} );
	}
}
```

`userId` 与 `users` 都是原始 `fetchUserData` 的输入，而且 `users` 还是输出。`safer_fetchUserData(..)` 接收这两个输入，并返回 `users`。为了确保我们没有在后面修改 `users` 时没有制造副作用，我们制造了一个 `users` 的本地拷贝。

这种技术几乎只有有限的用处，因为如果你不能把函数本身修改为纯粹的，那么你也不太可能修改它周围的代码。然而，在可能的情况下探索它一下还是有帮助的，因为它是我们的修改中最简单的一种。

不论这对于重构为纯函数来说是不是一种实际可行的技术，重点是函数的纯粹性只需要如皮毛一般肤浅。也就是，**一个函数的纯粹性是从外部判断的**，而不管内部发生了什么。只要一个函数的使用表现为纯粹的，那么它就是纯粹的。在一个纯函数内部，可以由于各种原因 —— 适度地！—— 使用非纯粹的技术，包括最常见的为了性能而这样做。它不一定是像人们说的那样，是“海龟背地球”。

但是要非常小心。程序中任何不纯粹的部分，即便它被一个纯函数包装而且仅为纯函数所用，也是潜在的 bug 以及代码读者困惑的源头。我们的总体目标是尽可能减少副作用，而不是仅将它们藏起来。

### Covering Up Effects

很多时候你都不能通过修改代码来将词法自由变量封装在一个包装函数的作用域中。例如，非纯函数存在于一个不可控的第三方库的文件中，包含这样一些东西：


```js
var nums = [];
var smallCount = 0;
var largeCount = 0;

function generateMoreRandoms(count) {
	for (let i = 0; i < count; i++) {
		let num = Math.random();

		if (num >= 0.5) {
			largeCount++;
		}
		else {
			smallCount++;
		}

		nums.push( num );
	}
}
```

在我们程序的其他部分使用这个工具时 *隔离* 侧因/副作用的粗暴策略是，创建一个执行下列步骤的接口函数：

1. 捕获将要被影响的当前状态
2. 设置初始输入状态
3. 运行这个非纯函数
4. 捕获副作用状态
5. 恢复原始状态
6. 返回捕获的副作用状态

```js
function safer_generateMoreRandoms(count,initial) {
	// (1) save original state
	var orig = {
		nums,
		smallCount,
		largeCount
	};

	// (2) setup initial pre-side effects state
	nums = initial.nums.slice();
	smallCount = initial.smallCount;
	largeCount = initial.largeCount;

	// (3) beware impurity!
	generateMoreRandoms( count );

	// (4) capture side effect state
	var sides = {
		nums,
		smallCount,
		largeCount
	};

	// (5) restore original state
	nums = orig.nums;
	smallCount = orig.smallCount;
	largeCount = orig.largeCount;

	// (6) expose side effect state directly as output
	return sides;
}
```

要使用 `safer_generateMoreRandoms(..)` 的话：

```js
var initialStates = {
	nums: [0.3, 0.4, 0.5],
	smallCount: 2,
	largeCount: 1
};

safer_generateMoreRandoms( 5, initialStates );
// { nums: [0.3,0.4,0.5,0.8510024448959794,0.04206799238...

nums;			// []
smallCount;		// 0
largeCount;		// 0
```

为了避免几个侧因/副作用要做许多手动工作；要是它们一开始就不存在就容易多了。但如果我们别无选择，那么为了在我们程序中避免意外这种额外的努力还是值得的。

**注意：** 这种技术其实只会在你对付同步代码时有效。异步代码不能用这种方式可靠地管理，因为它不能防止程序其他部分临时地访问/修改状态变量。

### Evading Effects

当我们要对付的副作用的性质是通过引用修改了一个直接输入值（对象，数组等等），我们同样可以创建一个接口函数来与之互动，从而取代原始的非纯函数。

考虑如下代码：

```js
function handleInactiveUsers(userList,dateCutoff) {
	for (let i = 0; i < userList.length; i++) {
		if (userList[i].lastLogin == null) {
			// remove the user from the list
			userList.splice( i, 1 );
			i--;
		}
		else if (userList[i].lastLogin < dateCutoff) {
			userList[i].inactive = true;
		}
	}
}
```

`userList` 数组本身，外加它里面的对象，都被修改了。防护这种副作用的一个策略是，首先进行一次深拷贝（好吧，只是不是浅拷贝）：

```js
function safer_handleInactiveUsers(userList,dateCutoff) {
	// make a copy of both the list and its user objects
	let copiedUserList = userList.map( function mapper(user){
		// copy a `user` object
		return Object.assign( {}, user );
	} );

	// call the original function with the copy
	handleInactiveUsers( copiedUserList, dateCutoff );

	// expose the mutated list as a direct output
	return copiedUserList;
}
```

这种技术的成功取决于你对值的 *拷贝* 进行得多彻底。这里 `userList.slice()` 不好用，因为它只创建了 `userList` 数组本身的一个浅拷贝。数组中的每一个元素都是一个需要被拷贝的对象，所以我们得额外地花些心思。当然，如果这些对象内部还有对象（有可能！），那么拷贝就需要更加健壮。

#### `this` Revisited

另外一种由引用引起的侧因/副作用是在 `this` 敏感的函数中使用 `this` 作为一种隐含的输入。关于为什么 `this` 关键字对 FP 程序员来说是个问题，详见第二章的“This是什么”。

考虑如下代码：

```js
var ids = {
	prefix: "_",
	generate() {
		return this.prefix + Math.random();
	}
};
```

我们的策略与前一节中的讨论类似：创建一个接口函数，强制 `generate()` 函数使用一个可预测的 `this` 上下文环境：

```js
function safer_generate(context) {
	return ids.generate.call( context );
}

// *********************

safer_generate( { prefix: "foo" } );
// "foo0.8988802158307285"
```

这些策略都不是天衣无缝的；对侧因/副作用的最安全的防护是不要产生它们。但如果你在试着改进你程序的可读性和信用等级，那么尽量减少侧因/副作用是向前迈进的一大步。

实质上，我们没有真正地消灭侧因/副作用，而是包容并限制它们，以使我们的代码更禁得住检验和可靠。如果稍后我们的程序出现 bug，那么我们就知道代码中依然使用侧因/副作用的部分最有可能是罪魁祸首。

## Summary

副作用对代码的可读性与质量是有害的，因为它们使你的代码更难于理解。副作用还是程序中最常见的 bug *起因* 之一，因为搬弄它们很困难。幂等性是一种通过创建实质上一次性的操作来限制副作用的策略。

纯函数是我们最好的避免副作用的方式。一个纯函数总是对相同的输入返回相同的输出，而且没有侧因或副作用。引用透明性进一步说明，一个出函数调用可以用它的输出替换 —— 更多地是思维上的行使而非真这么做 —— 而程序的行为不会有变化。

将一个非纯函数重构为纯函数是不错的选择。但如果那不可能，就可以封装侧因/副作用，或者创建一个纯粹的接口来防护它们。

没有程序是完全没有副作用的。但在实际中要在尽可能多的地方首选纯函数。尽可能多地将非纯函数的副作用集中在一起，这样如果 bug 发生，定位并检查嫌疑最大的祸首时会容易一些。
