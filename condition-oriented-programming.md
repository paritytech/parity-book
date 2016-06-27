# Condition-Orientated Programming

Condition-Orientated Programming (COP) is a hybrid approach between functional and imperative programming. Done properly it is a tool in your arsenal for writing safe, secure contracts. It helps make your contract code comprehensively auditable and - loosely speaking - informally provable to have correct run-time operation.

COP is not language specific; it is more of a loose methodology than particular syntax. However, with its function modifiers and events, it is particularly well-suited to the Solidity language.

Simply put, COP has just one main aim:

**Function bodies should have no conditional paths.**

Or alternatively:

**Never mix transitions with conditions.**

This may seem like a difficult goal to achieve for an imperative language, since conditional paths are how you achieve the rich state-transitions which allow interesting operational dynamics. To achieve it, we try to split all conditions apart from the state-transitions that they guard. We name each independently and combine to form real functions. 

The problem with such conditional paths within transition logic is that they add conceptual non-linearity over state semantics. Potential bugs hide when the programmer believes a conditional (and thus the state it projects onto) means one thing when in fact it means something subtly different.

A single-level conditional is bad enough, but when multi-level conditionals are introduced, the complexity (i.e. the paths which the programmer must consider in all states of the world) increases exponentially and it quickly becomes impossible to reason about the entire contract's state transitions without formal tools not generrally available.

COP addresses this by requiring the programmer to explicitly enumerate all such conditionals. Logic becomes flattened into non-conditional state-transactions. The condition fragments can then be properly documented, reused, reasoned-about and attributed with requirements and implications. Essentially, COP uses pre-conditions as a first-class citizen in programming.

## How it works

If you have already used Solidity, the chances are that you inadvertedly flirted with COP already. Let's look at a simple token contract:

```
contract Token {
	// The balance of everyone
	mapping (address => uint) public balances;

	// Constructor - we're a millionaire!
	function() {
		balances[msg.sender] = 1000000;
	}

	// Transfer `_amount` tokens of ours to `_dest`.
	function transfer(uint _amount, address _dest) {
		balances[msg.sender] -= _amount;
		balances[_dest] += _amount;
	}
}
```

The astute reader will realise there is a bug here: the `transfer` function doesn't ensure that the sender has enough in their account. The normal imperative-language fix for this would be to introduce an `if` statement:

```
function transfer(uint _amount, address _dest) {
	if (balances[msg.sender] >= _amount) {
		balances[msg.sender] -= _amount;
		balances[_dest] += _amount;
	}
}
```

Or perhaps:

```
function transfer(uint _amount, address _dest) {
	if (balances[msg.sender] < _amount)
		return;
	balances[msg.sender] -= _amount;
	balances[_dest] += _amount;
}
```

However both of these solutions rather miss the point of COP; we're muddling the implementation (which ever one we choose) with the *meaning*, which remains the same in both cases: that the executing account `msg.sender`, should have a balance of at least `_amount`. As a COP coder, we understand this problem perfectly since both solutions break our fundamental rule:

**Function bodies should have no conditional paths.**

So in COP, we rather *abstract* the condition (`balances[msg.sender] >= _amount`) and create a function modifier:

```
modifier only_with_at_least(uint x) { if (balances[msg.sender] >= x) _ }
```

This piece of code fundamentally abstracts the notion of "executing account has a balance of at least some particular amount". With it in place, we no longer need to think in terms of conditionals, and most importantly, we don't need to mix pre-condition logic with state-transition logic. This allows a far greater scope for human-understandable analysis of state-transitions.

Here's the new `transfer` function:

```
function transfer(uint _amount, address _dest) only_with_at_least(_amount) {
	balances[msg.sender] -= _amount;
	balances[_dest] += _amount;
}
```

## Abstraction and Reuse

Suppose we have another function, which allows anyone with a balance more than 1000 to vote on some issue. We'll assume for now that voting is just a case of setting the value of an `address`-indexed mapping.

In our old scheme of things, we'd have a function like this:

```
function vote(uint _opinion) {
	if (balances[msg.sender] >= 1000) {
		votes[msg.sender] = _opinion;
	}
}
```

Added to our old codebase, we would now have two similar-meaning conditionals. In principle, we would like to have only one such conditional, audited and documented once but used twice. With COP that's exactly what we do:

```
function vote(uint _opinion) only_when_at_least(1000) {
	votes[msg.sender] = _opinion;
}
```

This makes our `vote` function substantially more readable and allows us to reuse important guard-logic, minimisnig our potential attack surface.

## More complex transitions

By discouraging conditional paths from our state-transitions, we limit the complexity of our state-transitions. This hugely helps with auditing since it allows us apply the divide and conquer strategy to program logic analysis and independently check the logic of state transitions from the conditional logic on which they are gated. However sometimes the state transition itself includes gated logic internally.

Following on the voting example, suppose we extend the `transfer` function so that we ensure that any new, reduced, balance has no vote.

In the traditional, imperative, way we would simply place a conditional near the balance reduction code:

```
function transfer(uint _amount, address _dest) {
	if (balances[msg.sender] >= _amount) {
		balances[msg.sender] -= _amount;
		balances[_dest] += _amount;
		if (balances[msg.sender] < 1000) {
			votes[msg.sender] = 0;	// Clear their vote.
		}
	}
}
```

However, this rather goes against the grain of COP. However we cannot address this directly with a new function modifier since there is no obvious function to modify; we actually wish to place the guard within an internal scope of `transfer`. In this case (at least with Solidity), we rather create a new (inline) function:

```
function clear_undeserved_vote(account _who) only_with_under(1000) only_when_voted {
	delete votes[_who];
}
```

Note `inline` is not yet available in Solidity; we would use it here whenever it be available. This function relies on two modifiers which are easily coded (and audited):

```
modifier only_with_under(uint x) { if (balances[msg.sender] < x) _ }
modifier only_when_voted { if (votes[msg.sender] != 0) _ }
```

We can then use this function within our `transfer` function:

```
function transfer(uint _amount, address _dest) only_with_at_least(_amount) {
	balances[msg.sender] -= _amount;
	balances[_dest] += _amount;
	clear_undeserved_vote();
}
```

## Conclusion

The main part of our final contract has changed from:

```
contract Token
{
	//...

	function transfer(uint _amount, address _dest) {
		if (balances[msg.sender] >= _amount) {
			balances[msg.sender] -= _amount;
			balances[_dest] += _amount;
			if (balances[msg.sender] < 1000) {
				votes[msg.sender] = 0;	// Clear their vote.
			}
		}
	}

	function vote(uint _opinion) {
		if (balances[msg.sender] >= 1000) {
			votes[msg.sender] = _opinion;
		}
	}
}
```

To the new:

```
contract Token
{
	//...

	modifier only_with_at_least(uint x) { if (balances[msg.sender] >= x) _ }
	modifier only_with_under(uint x) { if (balances[msg.sender] < x) _ }
	modifier only_when_voted { if (votes[msg.sender] != 0) _ }

	function clear_undeserved_vote(account _who) only_with_under(1000) only_when_voted {
		delete votes[_who];
	}

	function transfer(uint _amount, address _dest) only_with_at_least(_amount) {
		balances[msg.sender] -= _amount;
		balances[_dest] += _amount;
		clear_undeserved_vote();
	}

	function vote(uint _opinion) only_when_at_least(1000) {
		votes[msg.sender] = _opinion;
	}
}
```

The code we have now forces the coder to document the internals, abstracts the important parts to ensure that no copy/paste bugs don't creep in, and has no forking in the execution path. It can be documented and audited, piece-by-piece in a comprehensive and methodical fashion. And, even if left undocuemnted, it is far more comprehensible, with the named conditions than the original version which muddles conditions with logic.

































