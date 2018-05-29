# A Case for Safe Eval
>“Eval is only one letter away from evil.”

— Js developers

>“Do not ever use `eval`!"

— MDN page on eval

Eval is the universally shunned function of the JS standard library. Folk programming logic tells us that any usage can be refactored away to code that doesn't rely on it, and that it should never be used on a source that you don't trust.  Even if you somehow find a use case that works and you sanitize the user input, it's said that by including eval in your code base, you will be encouraging future developers to use it for bad purposes. I'm going to attempt to refute these arguments by walking through my theoretical implementation of Safe Eval.
 
### Introducing Safe Eval
 Safe eval is a wrapper around eval where only certain characters are allowed to be executed and the rest are thrown away.  It's designed to only permit pure, no side effect algorithms that have to halt as well as no XSS. These characters are:
```js
[?:] [0-9.()] [+-/*] [><=!&|]
```

### Let’s evaluate what’s allowed with this set of characters:

### ```[?:]``` Ternary expressions

By using expressions like  `8 + 5 > 3 ? 5 : 3`, you can replicate almost any algorithm without iteration or recursion in it. 

### ```[0-9 . ([^\s]+) +-/*]``` Arithmetic

0-9 and arithmetic operators allow us to create constants and apply arithmetic operations on them. The `[^\s]+` means there needs to be at least one non-whitespace character in-between the parenthesis to avoid function calls.

### ```[><=!&|]``` Boolean Expressions

By using `<=, <, >, >=, ===, ==!, &&, ||`, we can evaluate any boolean expressions.

### Let’s evaluate what this can’t do:

1.  Call any function! It’s impossible to do so. Although you can construct a regex with this set of characters, you can’t access properties on the regex with just numbers. Additionally, `(() => 5)()` doesn't work because we don't allow open and close parens next to each other.
2.  Modify any variables! There’s no way to get a reference to a variable with this set of characters. No variable name can be made up with this set of characters.
3.  Loop infinitely! There’s no way to recurse, while, or any such thing. This formula is guaranteed to halt. It’s possible to make a formula that will take a very long time to evaluate, but that’s it! You could also set a maxlength on the formula, so as to make this very difficult. I would love to see what short character equations are possible with this set that take more than a few milliseconds to evaluate.
4.  Make a string, object, or array. Unfortunately, by allowing  quotes, square brackets, or curly brackets, you can most likely write any code that you want at this point. See [jsfuck](http://www.jsfuck.com/) for example. 

So Safe Eval is safe! You can confidently run code from users by restricting their  JS to a small, purely mathematical instruction set.

You might be thinking at this point:
>Well great, I guess it’s somewhat safe to run eval on a purely mathematical expression. But why would I want to do that? There is no input to the expression, so why not just serve up the result? Why calculate a formula on the client at all?

Arithmetic eval is actually the second step of the function I’m proposing. The first step is providing inputs into the formula via interpolation.

>What! Why would you ever need to do that! There's no valid use case for this!

### A valid use case for Interpolation into Safe Eval:

User Marsha is customizing their shop for their hard earned neopets loot. They want to encourage loyalty, so they want to have a discount if a customer is a repeat buyer. She selects "add a dynamic discount" from the shop edit screen and she is greeted with a form like this one:
|| Create a Discount |
|--|--|
| Variables (\<select>) |`price`, `numberPreviousItemsPurchased`, `affinityForCats`, `isActuallyARobot` |
|Formula (\<text>)|*e ^(i * pi) + 1 === 0*|

They select price and numberPreviousItemsPurchased as their variables.
They fill in the Formula field like so:
```js
price  -
price  *
	(numberPreviousItemsPurchased  >  5) ?
		0.05 :
		numberPreviousItemsPurchased  *  0.01
```
Which then gets saved to the backend like this:

```js
{
	formula: “price  - price  * (numberPreviousItemsPurchased  >  5) ? 0.05 :
numberPreviousItemsPurchased  *  0.01”,
	dependentKeys: [‘price’, ‘numberPreviousItemsPurchased’]
}
```
Then, after the store updates, a returning user visits the site. Ed, who has bought 3 golden pet eggs from the store previously, has a previousItemsPurchased value of 3. When Ed looks at prices, they see a discount on everything for 3%! 

### How is this calculated?

When the store renders prices for each item, it checks to see if there is a discount, then it gets the formula, interpolates the variables using data from the `dataStore` and calculates the resulting price.

Interpolation looks like this:

```js
function interpolate(formula, dependentKeys, dataStore){
	return dependentKeys.reduce((acc, key) => {
		let value = dataStore[key]; //Lookup the variable
		return acc.replace(RegExp(key, 'g'), value);
		//Replace all occurrences of the string with the value of the variable
	}, formula);
};
interpolate("price * .99", ["price"], { price: "100" });
// returns "100 * .99"
```
Calculation looks like this:
```js
function safeEval(expression){
	const toEval = expression.replace(/(\(\s*\)|[^0-9.()+\-*\/><=!&|?:])+/g, ‘’);
	// ^ Removes unsafe chars (including ( ), but not (5 + 5))
	// See https://regex101.com/r/Pt82Gi/3 for examples.
	try {
		return eval(toEval);
	} catch(error) {
		return null;
		//Rather than try to recover, just return.
	}
}
safeEval("100 * .99"); //returns 99
```
Also, change detection has to be implemented as well, such that if Ed buys a 4th item, the price would know to update. Most modern web frameworks feature some way of triggering recalculation of values if you update dependent keys, so I’ll leave that code out.

I think this is an amazing solution as far as giving your users flexibility and for lessening the burden of extensibility on developers.

### Pros
1.  Powers users will love this - they can tweak their formulas to their heart’s content. They can even create new ways of discounting that the developer didn’t think of!
2.  To add functionality to it, the only thing the developer has to do is add more variables! Rather than having to make a complicated logic for each time that a new feature for more interesting promotions, they can add variables and let the user figure out whatever promotion they want.
3.  For less experienced users, the developer can provide preset formulas and prices, and allow the user to fill in these widgets, which will be transformed into dynamic formulas by the client. Think ‘buy one get one free’ or ‘dollar off’ discount creation. 

### Cons
1.  Users may accidentally make promotions that result in way more discount than they’re comfortable with. This can be mitigated by preventing users from uploading promotions that cause negative prices, or checking in with the user when their promotions pass a limit. However, one of the best mitigations for this is to have real time feedback on what the store will look like when a user is creating a promotion, so they can see if their promo will have unwanted consequences.
2.  Customers might end up in scenarios where pricing is confusing to them. Imagine a power user who makes a masterpiece of a formula, but fails to explain it adequately. The customer navigating the site might be completely confused as to why the prices are fluctuating so much, or how to pay for things the best way. This is more of a failing on the user than the system - power users have to learn the lesson that complicated pricing will cause problems.
3.  If there are too many overly complicated discounts on the page, it might cause the computer to take a long time to render the discounts. This is something the store owner should be incentivized not to do if they want to make money, and customers can simply avoid slow loading stores. Additionally, slow loading stores can be pushed to the bottom of search results.

At this point, you may be thinking,
>“Well you don’t need eval to do that. You could do the exact same implementation using a calculator, and it wouldn't be so unsafe!"

### Ok, let’s build a calculator.

My calculator will only support addition and will assume that input is in prefix, for simplicities sake.  First we have to build a lexer for the calculator so we can operate symbolically.
```js
//lexer.js
function  lex(raw) {
  const  stack  = [];
  const  lexed  = [];
  raw.split('  ').forEach(symbol => {
    if  (/[+]/.test(symbol)) {
      lexed.push(
        stack.pop(),
        stack.pop(),
        { type: 'operator', val: '+' }
       );
      return;
    }
    if  (/[0-9]+/.test(symbol)) {
      stack.push({ type: 'number', val: symbol });
      return;
    }
    if  (/[a-zA-Z]+/.test(symbol)) {
      stack.push({ type: 'variable', val: symbol });
      return;
    }
  });
  return { type: 'program', val: lexed };
}
const  parsedFormula  =  lex('price 5 +');
console.log(parsedFormula);
/* prints: 
{
  type: 'program',
  val: [
    { type: 'number', val: 5 },
    { type: 'variable', val: 'price' },
    { type: 'operator', val: '+' }
  ]
}
*/
```
Then we create a calculator that gets the numeric value by recursing through the tree.
```js
function  calculate(expression, stack, dataStore) {
  const  val  =  0;
  switch  (expression.type) {
    case  'program':
      return  expression.val.reduce(
        (sum, exp)  =>  sum  +  calculate(exp, stack, dataStore),
        val
      );
    case  'number':
      stack.push(expression.val);
    case 'operator':
      return  val  +  handleOperator(stack, expression.val);
    case  'variable':
      stack.push(dataStore[expression.val]);
  }
  return  val;
}
function  handleOperator(stack, val) {
  switch  (val) {
    case  '+':
      return  parseInt(stack.pop())  +  parseInt(stack.pop());
  }
}
calculate(parsedFormula, [], { price: 5 }); // 10
```

### This is really complicated!

This is very quickly becoming a huge pain for me to write. I have to write a compiler, and a lexer. I have to continue extending this for each new kind of operator added. Just by looking at the value returned from the backend, it’s hard for me to parse what the formula is supposed to be. Testing, which will be a necessity, is going to require a lot of work to make sure that it covers all edge cases.

Most importantly, it’s blatant duplication of code that already exists on the client. I’m essentially rewriting the browsers implementation of parsing JS and evaluating it. Their implementation is going to be 1000 more resilient, more performant, and more stable than anything I could write.

The fact is, using `eval` instead of writing a lexer/calculator is a much better option for both software engineering concerns, and performance. Implementing a lexer/calculator is about as difficult and slow as implementing a reduced set of JS in JS.

### Counterarguments against code quality

>"Well eval can never be safe! Even if you only allow arithmetic in your eval, some junior developer will come by and make a change and break your implementation, causing XSS bugs galore!"

Your junior (and senior) developers are writing XSS bugs right now, without using eval at all.

Eval is unsafe, but so is writing to the DOM without unescaping. We know that it's dangerous, but very frequently we do that - for example allowing a user to have a link in their bio, or when a user uploads an image to a server and then it is rendered. Both of these inputs are sanitized, so we can be sure that they won't cause a vulnerability. Markdown, which this blog post is written in, is compiled to HTML, sanitized, and then injected into the DOM.

If we can find a way to write a sanitizer for Markdown, then we can find a way to write a sanitizer for Safe Eval. It's true that it needs tests, it will need to be battle hardened and it will need to be an open source project that people can trust.

>"But even if you did have an open source project that was trusted, people would still break it accidentally."

As long as maintainers don't permit forbidden characters, no XSS *should* be possible. Over time, maintainers may want to permit features like `Math.*` and similar functions, but they would only get to do so very carefully! Changes that allow more character sets would need meticulous security audits before they could get merged into new releases.

>"Eval encourages developers to use eval everywhere, and that's going to cause problems!"

Since Safe Eval lives in a library, you can still have style rules that prevent accepting PRs with eval in them. I strongly recommend that you do not allow people to submit pull requests with eval. If you want an automated way of doing this, I recommend setting up eslint with the `no-eval` rule and having a policy of not merging PRs that fail eslint.

### Eval can be safe.

Eval is one of the most notorious functions in the JS standard library, however, I don't think that means we should ban it to strange edge cases related to importing code. Letting users input code into your website is an amazing feature that gives them all of the options that a programming language has, and it can be very dangerous for that reason. However, by removing all of the potentially dangerous bits of a programming language, we're still left with a feature that is very flexible and completely safe. I've demonstrated that there is at least one valid use case for Safe Eval and that it is a feature which gives the user the ability to be creative as well as less work for the developer. Although you can achieve the same feature set without using Eval, you have to do so with a very complicated, slow lexer and calculator. Additionally, by using Safe Eval frin a library, you avoid having to put eval in your code and you avoid any potentially dangerous usages of eval.
> I still don't think eval is a good idea!

I'd love to hear why you think that! Tweet at me @realRobWebb or you can open an issue on this essay [here.](https://github.com/robert-j-webb/stackedit)

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgwMjQ2OTAwNCwxNTMxOTE1Mjc3LDEyNz
A4NDA2MjQsNTU1MTcwODE4LDE5MDM4MjgzNjEsLTYwMDkyNjky
MCwtODY4NjY2Mzg3LDM3NzE3MzkyNywtNDExMzY3MTE2LDkzOT
U0ODk5OSwtMTY3MzIzNzIxOCwtMjYwMjUzMzY3LC0xNzg0MTQx
NDc5LC0zMzA4MzI1OTYsMTQ5MjE0MDM3OSwtMTEwMzIyNzA0OC
wxMzExODE5NDQ5LC0xNzU2NDQ3ODA3LC0xMjQxMDc1Mjg5LDE2
MDU0OTMwMTFdfQ==
-->