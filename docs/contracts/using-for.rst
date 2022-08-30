.. index:: ! using for, library

.. _using-for:

*********
Using For
*********

The directive ``using A for B;`` can be used to attach functions (``A``)
as operators to user-defined types or as member functions to any type (``B``).
The member functions receive the object they are called on as their first
parameter (like the ``self`` variable in Python). The operator functions
receive operands as parameters.

It is valid either at file level or inside a contract,
at contract level.

The first part, ``A``, can be one of:

- a list of file-level or library functions (``using {f, g, h, L.t} for uint;``) -
  only those functions will be attached to the type as member functions,
- the name of a library (``using L for uint;``) -
  all functions (both public and internal ones) of the library are attached to the type
  as member functions,
- a list of assignments of functions to operators (``using {f as +, g as -} for T;``) -
  the functions will be attached to the type (``T``) as operators.

At file level, the second part, ``B``, has to be an explicit type (without data location specifier).
Inside contracts, you can also use ``using L for *;``, which has the effect that all functions
of the library ``L`` are attached to *all* types.

If you specify a library, *all* functions in the library are attached,
even those where the type of the first parameter does not
match the type of the object. The type is checked at the
point the function is called and function overload
resolution is performed.

If you use a list of functions (``using {f, g, h, L.t} for uint;``),
then the type (``uint``) has to be implicitly convertible to the
first parameter of each of these functions. This check is
performed even if none of these functions are called.

If you define an operator for a user-defined type (``using {f as +} for T``), then
the type (``T``), types of function parameters and a type of function return value
have to be the same. The type (``T``) does not include data location.
But, data location of the function parameters and function return value must be
the same.

The ``using A for B;`` directive is active only within the current
scope (either the contract or the current module/source unit),
including within all of its functions, and has no effect
outside of the contract or module in which it is used.

When the directive is used at file level and applied to a
user-defined type which was defined at file level in the same file,
the word ``global`` can be added at the end. This will have the
effect that the functions are attached to the type everywhere
the type is available (including other files), not only in the
scope of the using statement.

Let us rewrite the set example from the
:ref:`libraries` section in this way, using file-level functions
instead of library functions.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.13;

    struct Data { mapping(uint => bool) flags; }
    // Now we attach functions to the type.
    // The attached functions can be used throughout the rest of the module.
    // If you import the module, you have to
    // repeat the using directive there, for example as
    //   import "flags.sol" as Flags;
    //   using {Flags.insert, Flags.remove, Flags.contains}
    //     for Flags.Data;
    using {insert, remove, contains} for Data;

    function insert(Data storage self, uint value)
        returns (bool)
    {
        if (self.flags[value])
            return false; // already there
        self.flags[value] = true;
        return true;
    }

    function remove(Data storage self, uint value)
        returns (bool)
    {
        if (!self.flags[value])
            return false; // not there
        self.flags[value] = false;
        return true;
    }

    function contains(Data storage self, uint value)
        view
        returns (bool)
    {
        return self.flags[value];
    }


    contract C {
        Data knownValues;

        function register(uint value) public {
            // Here, all variables of type Data have
            // corresponding member functions.
            // The following function call is identical to
            // `Set.insert(knownValues, value)`
            require(knownValues.insert(value));
        }
    }

It is also possible to extend built-in types in that way.
In this example, we will use a library.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.13;

    library Search {
        function indexOf(uint[] storage self, uint value)
            public
            view
            returns (uint)
        {
            for (uint i = 0; i < self.length; i++)
                if (self[i] == value) return i;
            return type(uint).max;
        }
    }
    using Search for uint[];

    contract C {
        uint[] data;

        function append(uint value) public {
            data.push(value);
        }

        function replace(uint from, uint to) public {
            // This performs the library function call
            uint index = data.indexOf(from);
            if (index == type(uint).max)
                data.push(to);
            else
                data[index] = to;
        }
    }

Note that all external library calls are actual EVM function calls. This means that
if you pass memory or value types, a copy will be performed, even in case of the
``self`` variable. The only situation where no copy will be performed
is when storage reference variables are used or when internal library
functions are called.

Another example shows how to define a custom operator for a user-defined type:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.17;

    using {add as +} for Point;

    struct Point {
        uint x;
        uint y;
    }

    function add(Point memory a, Point memory _b) pure returns (Point memory result) {
        result.x = a.x + b.x;
        result.y = a.y + b.y;
    }

    contract C {
        function test() pure public {
            Point memory p1 = Point({x: 3, y: 4});
            Point memory p2 = Point({x: 7, y: 16});

            Point memory result = p1 + p2;
            require(result.x == 10);
            require(result.y == 20);
        }
    }
