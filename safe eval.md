# A Case for Safe Eval

>“Eval is only one letter away from evil.”

— Js developers
>“Eval is only to be used for loading code, nothing else.”

— JavaScript style standards

JavaScript developers everywhere agree that eval is never to be used in production code because it’s unsafe, it’s hacky, and it can lead to unpredictable behavior that’s difficult to debug. I am not disputing that about eval, however, a safe eval could be made that has none of these problems. Safe eval is a wrapper around eval where only certain characters are allowed to be executed, the rest are thrown away. These characters are:
```js
[0-9.()] [+-/*] [><=!] [&|] [?:]
```
### Let’s evaluate what’s allowed with this set of characters:
### ```[?:]``` Ternary expressions
By using expressions like  8 + 5 > 3 ? 5 : 3, you can replicate almost any algorithm without iteration or recursion in it. By avoiding looping or recursion, we have a guarantee that the formula supplied will halt.
### ```[0-9 . () +-/*]``` Arithmetic
0-9 and arithmetic operators allow us to make formulas and expressions that are arithmetic, like describing a distance or a range. We don’t have access to the Math functions, but we can still emulate many of them by reducing them into their elementary forms. By avoiding allowing the Math function, we don’t have to worry about the dangers of allowing strings or characters to be parsed.
### ```[><=! &|]``` Boolean Expressions

By using `<=, <, >, >=, ===, ==!, &&, ||`, we can evaluate whichever boolean expressions we need or would want access to.

### Let’s evaluate what this can’t do:
1.  Call any function! It’s impossible to do so. Although you can construct a regex with this set of characters, you can’t access properties on the regex with just numbers.
2.  Modify any variables! There’s no get a reference to a variable with this set of characters. No variable name can be made up with just this set of characters.
3.  Loop infinitely! There’s no way to recurse, while, or any such thing. This formula is guaranteed to halt. It’s possible to make a formula that will take a very long time to evaluate, but that’s it! You could also set a maxlength on the formula, so as to make this very difficult. I would love to see what short character equations are possible with this set that take more than a few milliseconds to evaluate.
4.  Make a string, object, or array. Unfortunately, by allowing a quote, square bracket, or curly bracket, you can most likely write any code that you want at this point. See [jsFuck](http://www.jsfuck.com/) for example. Also, this makes the possible formula much more complicated, and therefore more difficult to debug.

You might be thinking at this point, well great, I guess it’s somewhat safe to run eval on a purely arithmetic expression. But why would I want to do that? There is no input to the expression, so why not just serve up the result? Why calculate a formula on the client at all?

Arithmetic eval is actually the second step of the function I’m proposing. The first step is providing inputs into the formula via interpolation.

### Let’s consider a scenario like this:

User Marsha is making a shop for their hard earned neopets loot. They want to encourage loyalty, so they want to have a discount if a customer is a repeat buyer. They go into their shop editor and they select, `add a discount to all products.` Then, rather than selecting a flat rate like just 1%, they instead indicate they want to use a formula. Then, from a list of variables, they select `number of previous items purchased.` Then, they are provided with this screen to enter their prices in:

```js
function (numberPreviousItemsPurchased, price) {
	// add your code here
}
```
They fill in the field like so:
```js
function (numberPreviousItemsPurchased, price) {
	if  (price  <  100) {
		return  price;
	}
	let  discount  = numberPreviousItemsPurchased  >  5
		?  0.05
		:  numberPreviousItemsPurchased  *  0.01;
	return  price  -  price  *  discount;
}
```
(This step might be pretty tricky to implement) This gets compiled into:
```js
//This is a pretty printed version of it
price  <  100
	?  price
	:  price  -
	   price  *
		(numberPreviousItemsPurchased  >  5
			?  0.05
			:  numberPreviousItemsPurchased  *  0.01);
```
Which then gets saved to the backend like this:

```js
{
	//The spaces would be stripped here, but it makes it less readable
	formula: “price<100 ? price : price - price * (numberPreviousItemsPurchased > 5 ? .05 : numberPreviousItemsPurchased * .01)”,
	dependentKeys: [‘price’, ‘numberPreviousItemsPurchased’],
	type: ‘discount’
}
```
They submit. They then update the copy on their store to feature this loyalty program, so that they can bring in more returning users.

Then, after the store updates, a returning user visits the site. Ed has bought 3 ties from the store previously, so their previousItemsPurchased is 3. When Ed looks at prices, they see a discount on everything over 100 neopoints for 3%! How is this calculated?
When the store renders prices for each item, it checks to see if there is a discount, then it gets the formula, interpolates the variables and calculates the resulting price.

Interpolation looks like this:

```js
function interpolate(formula, dependentKeys, dataStore){
	return dependKeys.reduce((acc, key) => {
		let value = dataStore.get(key); //Lookup the variable
		acc.replace(key, value, ‘g’); //Replace all occurrences of the string with the value of the variable
		return acc;
	}, formula);
};
```
Calculation looks like this:
```js
function safeEval(expression){
	const toEval = expression.replace(/[^0-9.()+\-*\/><=!&|?:]*/g, ‘’);
	return eval(toEval);	
}
```
Also, change detection has to be implemented as well, such that if Ed buys a 4th item, the price would know to update. Most modern web frameworks feature some way of triggering recalculation of values if you update dependent keys, so I’ll that code out.

I think this is an amazing solution as far as giving your users flexibility and for lessening the burden of extensibility on developers. Here are some pros that come to mind.

1.  Powers users will love this - they can tweak their formulas to their heart’s content. They can even create new ways of discounting that the developer didn’t think of!
2.  To add functionality to it, the only thing the developer has to do is add more variables! Rather than having to make a complicated logic for each time that a new feature for more interesting promotions, they can add variables and let the user figure out the new promotion.
3.  For less experienced users, the developer can provide preset formulas and prices, and allow the user to fill in these widgets, which will be reduced to dynamic formulas by the client. Think ‘buy one get one free’ or ‘dollar off’ discount creation.

  

Some cons:

  

1.  Users may accidentally make promotions that result in way more discount than they’re comfortable with. This can be mitigated by preventing users from uploading promotions that cause negative prices, or checking in with the user when their promotions pass a limit. However, one of the best mitigations for this is to have real time feedback on what the store will look like when a user is creating a promotion, so they can see in real time how things are affected.
2.  Customers might end up in scenarios where pricing is confusing to them. Imagine a power user who makes a masterpiece of a formula, but fails to explain it adequately. The customer navigating the site might be completely confused as to why the prices are fluctuating so much, or how to pay for things the best way. This is more of a failing on the user than the system - power users have to learn the lesson that complicated pricing will cause problems.
3.  If there are too many overly complicated discounts on the page, it might cause the computer to take a long time to render the discounts. This is something the store owner should be incentivized not to do if they want to make money, and customers can simply navigate back. Additionally, slow loading stores can be pushed to the bottom of search results.

  

  

At this point, you may be thinking, “Well you don’t need eval to do that. You could do it by building a calculator, and it wouldn’t be unsafe!”

  

Ok, let’s build a calculator. First we have to build a lexer for the calculator so we can operate symbolically. Let’s assume that the backend lexes for us, and it returns something like this for the formula:

  

```js

{

formula: {

program: {

type: 'program',

val: [

{ type: 'number', val: 5 },

{ type: 'number', val: 5 },

{ type: 'operator', val: '+' }

]

}

}

};

```js

function calculate(expression, stack) {

let val = 0;

switch (expression.type) {

case 'program':

val = expression.val.reduce((sum, exp) => sum + calculate(exp, stack), 0);

break;

case 'number':

stack.push(expression.val);

return 0;

case 'operator':

return handleOperator(stack, expression.val);

}

return val;

}

  

function handleOperator(stack, val) {

const val1 = stack.pop();

const val2 = stack.pop();

switch (val) {

case '+':

return val1 + val2;

// do other operators

}

}

```

  

This is very quickly becoming a huge pain point for me to write. I have to write complicated tests for this. I have to write a compiler, and a lexer. I have to continue extending this for each new kind of operator added. Just by looking at the value returned from the backend, it’s hard for me to parse what the formula is supposed to be. It’s no longer human readable in memory.

  

Most importantly, it’s blatant duplication of code that already exists on the client. I’m essentially rewriting the browsers implementation of parsing JS and evaluating it. Their implementation is going to be 1000 more resilient, more performant, and more stable than anything I could write.

  

Here’s the thing that bothers me the most about calling ‘eval’ unsafe - every time a developer lets unescaped html go into the Dom, they’re basically calling eval on that code. It’s true that it’s dangerous to write to the DOM unescaped, but very frequently we just have to do that - for example showing images that the user has uploaded, or embedding HTML that’s been sufficiently sanitized by the server. Every time you read a comment on GitHub, or look at an image on twitter, that comes from unescaping data and injecting it directly into the dom.

  

The fact is, a reduced character set eval is just as safe as rendering an image that a user uploads. If you sanitize the URL, you will be fine. If you don’t, you will have an XSS vulnerability. As developers, we have to be cautious of allowing users ability to add data to our site, but we don’t need to be afraid of it.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA2NzYxMDg4NiwxODM0MzgwOCwtNzcyOT
k4MzczLC05MDg1MDExNTksLTE5NDg2MjQ4OTNdfQ==
-->