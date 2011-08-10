function Y(outer)
{
	return
	(function(func)
	{
		return func(func);
	})
	(function(func)
	{
		return outer(
			function(arg)
			{
				return func(func)(arg);
			}
		);
	});
}

function factorial_recursive(int n)
{
	if(n == 0) return 1;
	else return n * factorial_recursive(n - 1);
}

function main()
{
	var factorial_y = function(var func)
	{
		return function(int n)
		{
			if(n == 0) return 1;
			else return n * func(n - 1);
		};
	};

	say("Recursive factorial gives: ", factorial_recursive(6));
	say("Y-Combinator factorial gives: ", Y(factorial_y)(6));
}

function Y(var outer) {
    return (->(func) func(func))(->(func) outer(->(arg) func(func)(arg)));
}

function main[main]()
{
    Y(->(func) ->(int n) n == 0 ? 1 : n * func(n - 1));
}
