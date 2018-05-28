# A Case for Safe Eval
>“Eval is only one letter away from evil.”

— Js developers
>“Do not ever use `eval`!"

— MDN page on eval

Eval is the universally shunned function of the JavaScript standard library. Folk programming logic tells us that any usage can be refactored away to code that doesn't rely on it, and that it should never be used on a source that you don't trust.  Even if you somehow find a use case that works and you sanitize the user input, it's said that by including eval in your code base, you will be encouraging future developers to use it for bad purposes. I'm going to attempt to refute these arguments by walking through my theoretical implementation of Safe Eval.
 
### Introducing Safe Eval
 Safe eval is a wrapper around eval where only certain characters are allowed to be executed and the rest are thrown away. These characters are:
```js
[0-9.()] [+-/*] [><=!] [&|] [?:]
```
### Let’s evaluate what’s allowed with this set of characters:
### ```[?:]``` Ternary expressions
By using expressions like  `8 + 5 > 3 ? 5 : 3`, you can replicate almost any algorithm without iteration or recursion in it. 
### ```[0-9 . () +-/*]``` Arithmetic
0-9 and arithmetic operators allow us to make formulas and expressions that are arithmetic, like describing a distance or a range. 
### ```[><=! &|]``` Boolean Expressions

By using `<=, <, >, >=, ===, ==!, &&, ||`, we can evaluate any boolean expressions.

### Let’s evaluate what this can’t do:
1.  Call any function! It’s impossible to do so. Although you can construct a regex with this set of characters, you can’t access properties on the regex with just numbers.
2.  Modify any variables! There’s no get a reference to a variable with this set of characters. No variable name can be made up with this set of characters.
3.  Loop infinitely! There’s no way to recurse, while, or any such thing. This formula is guaranteed to halt. It’s possible to make a formula that will take a very long time to evaluate, but that’s it! You could also set a maxlength on the formula, so as to make this very difficult. I would love to see what short character equations are possible with this set that take more than a few milliseconds to evaluate.
4.  Make a string, object, or array. Unfortunately, by allowing a quote, square bracket, or curly bracket, you can most likely write any code that you want at this point. See [jsfuck](http://www.jsfuck.com/) for example. 

You might be thinking at this point:
>Well great, I guess it’s somewhat safe to run eval on a purely arithmetic expression. But why would I want to do that? There is no input to the expression, so why not just serve up the result? Why calculate a formula on the client at all?

Arithmetic eval is actually the second step of the function I’m proposing. The first step is providing inputs into the formula via interpolation.

>What! Why would you ever need to do that! There's no valid use case for this!

### A valid use case for Safe Eval:

User Marsha is customizing their shop for their hard earned neopets loot. They want to encourage loyalty, so they want to have a discount if a customer is a repeat buyer. Marsha wants to add a discount of 1% for every item the customer has purchased, up to 5% off. Marsha happens to know some java

```js
function (numberPreviousItemsPurchased, price) {
	// add your code here
}
```
They fill in the field like so:
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
	//The spaces would be stripped here, but it makes it less readable
	formula: “price  - price  * (numberPreviousItemsPurchased  >  5) ? 0.05 :
numberPreviousItemsPurchased  *  0.01”,
	dependentKeys: [‘price’, ‘numberPreviousItemsPurchased’],
	type: ‘discount’
}
```
They submit. They then update the copy on their store to feature this loyalty program, so that they can bring in more returning users.

Then, after the store updates, a returning user visits the site. Ed has bought 3 ties from the store previously, so their previousItemsPurchased is 3. When Ed looks at prices, they see a discount on everything for 3%! 
### How is this calculated?
When the store renders prices for each item, it checks to see if there is a discount, then it gets the formula, interpolates the variables and calculates the resulting price.

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
	const toEval = expression.replace(/[^0-9.()+\-*\/><=!&|?:]*/g, ‘’);
	try {
		return eval(toEval);
	} catch(error) {
		return null; //Rather than try to recover, just return.
	}
}
safeEval("100 * .99"); //returns 99
```
Also, change detection has to be implemented as well, such that if Ed buys a 4th item, the price would know to update. Most modern web frameworks feature some way of triggering recalculation of values if you update dependent keys, so I’ll leave that code out.

I think this is an amazing solution as far as giving your users flexibility and for lessening the burden of extensibility on developers.

### Pros
1.  Powers users will love this - they can tweak their formulas to their heart’s content. They can even create new ways of discounting that the developer didn’t think of!
2.  To add functionality to it, the only thing the developer has to do is add more variables! Rather than having to make a complicated logic for each time that a new feature for more interesting promotions, they can add variables and let the user figure out the new promotion.
3.  For less experienced users, the developer can provide preset formulas and prices, and allow the user to fill in these widgets, which will be reduced to dynamic formulas by the client. Think ‘buy one get one free’ or ‘dollar off’ discount creation. 

### Cons
1.  Users may accidentally make promotions that result in way more discount than they’re comfortable with. This can be mitigated by preventing users from uploading promotions that cause negative prices, or checking in with the user when their promotions pass a limit. However, one of the best mitigations for this is to have real time feedback on what the store will look like when a user is creating a promotion, so they can see if their promo will have negative consequences.
2.  Customers might end up in scenarios where pricing is confusing to them. Imagine a power user who makes a masterpiece of a formula, but fails to explain it adequately. The customer navigating the site might be completely confused as to why the prices are fluctuating so much, or how to pay for things the best way. This is more of a failing on the user than the system - power users have to learn the lesson that complicated pricing will cause problems.
3.  If there are too many overly complicated discounts on the page, it might cause the computer to take a long time to render the discounts. This is something the store owner should be incentivized not to do if they want to make money, and customers can simply navigate back. Additionally, slow loading stores can be pushed to the bottom of search results.

At this point, you may be thinking,
>“Well you don’t need eval to do that. You could do it by building a calculator, and it wouldn’t be unsafe!”

### Ok, let’s build a calculator.

First we have to build a lexer for the calculator so we can operate symbolically. Let’s assume that the backend lexes for us, and it returns something like this for the formula:
```js
let originalFormula = '5 + 5';
console.log(lex(originalFormula));
{
	type: 'program',
	val: [
		{ type: 'number', val: 5 },
		{ type: 'number', val: 5 },
		{ type: 'operator', val: '+' }
	]
};
```
Then we create a reducer that gets the numeric value by recursing through the tree.
```js
function calculate(expression, stack) {
	let val = 0;
	switch (expression.type) {
		case 'program':
			return expression.val.reduce((sum, exp) => sum + calculate(exp, stack), val);
		break;
		case 'number':
			stack.push(expression.val);
			return val;
		case 'operator':
			return val + handleOperator(stack, expression.val);
	}
	return val;
}

function handleOperator(stack, val) {
	const val1 = stack.pop();
	const val2 = stack.pop();
	switch (val) {
		case '+':
			return val1 + val2;
		// Imagine other operators here
	}
}

calculate(parsedFormula, []) === 10; //true
```
### This is really complicated!
This is very quickly becoming a huge pain for me to write. I have to write complicated tests for this. I have to write a compiler, and a lexer. I have to continue extending this for each new kind of operator added. Just by looking at the value returned from the backend, it’s hard for me to parse what the formula is supposed to be.

Most importantly, it’s blatant duplication of code that already exists on the client. I’m essentially rewriting the browsers implementation of parsing JS and evaluating it. Their implementation is going to be 1000 more resilient, more performant, and more stable than anything I could write.

>"Well eval can never be safe! Even if you only allow arithmetic in your eval, some junior developer will come by and make a change and break your implementation, causing XSS bugs galore!"

### Your junior (and senior) developers are writing XSS bugs right now, without the help of eval.

Here’s the thing that bothers me the most about calling ‘eval’ unsafe is that every time a developer lets unescaped html go into the DOM, they’re basically calling eval on that code. It’s true that it’s dangerous to write to the DOM unescaped, but very frequently we do that - for example allowing a user to have a link in their bio, or when a user uploads an image to a server and you serve it. Markdown, which this blog post is written in, is compiled to HTML, which is then sanitized by the server and then injected into the DOM.

If we can find a way to write a sanitizer for Markdown, then we can find a way to write a sanitizer for Safe Eval. It's true that it needs tests, it will need to be battle hardened and it will need to be an open source project that people can trust.

>"But even if you did have an open source project that was trusted, people would still break it accidentally.

Tests should stop it from breaking. Basically just by reproducing each possible XSS in it's own test for eval, you should be able to prevent future devs from accidentally causing XSS.

>"Eval encourages developers to use eval everywhere, and that's going to cause problems!"

Since Safe Eval lives in a library, you can still have style rules that prevent accepting PRs with eval in them. In fact, I 100% recommend that you do not allow people to submit pull requests with eval. If you're incapable of enforcing this rule on your team, I recommend not merging PRs that fail to pass eslint as a rule.

### Summary

I don't quite know what to say here

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTAwNTIyMDI5NiwyMDE2OTEyODQzLC05Nj
k1MzU0ODcsMjEyODQ5NDAwXX0=
-->