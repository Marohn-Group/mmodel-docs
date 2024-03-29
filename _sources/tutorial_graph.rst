Build a model graph
=============================

A directed acyclic graph (DAG) is a directed graph without any cycles.
The model graphs in *mmodel* are based on the DAG, where each node represents
an execution step, and each edge represents the data flow from one callable
to another. DAG structure allows us to create model graphs with nonlinear
nodes.

Define a graph
--------------

The ``Graph`` class is the main graph class to establish a model graph.
The class inherits from ``networkx.DiGraph``, which is compatible with all
*NetworkX* operations
(see `documentation <https://networkx.org/documentation/stable/>`_).
To create and modify the graph,
see the documentation for adding 
`nodes <https://networkx.org/documentation/stable/tutorial.html#nodes>`_
and adding `edges <https://networkx.org/documentation/stable/tutorial.html#edges>`_.

Aside from the *NetworkX* operations,
*mmodel* provides ``add_grouped_edge`` and ``add_grouped_edges_from`` to add edges.

.. code-block:: python

    from mmodel import Graph, Node
    
    G = Graph()

    G.add_grouped_edge(['a', 'b'], 'c')

    # equivalent to
    # G.add_edge('a', 'b')
    # G.add_edge('a', 'c')

Similarly, with multiple grouped edges

.. code-block:: python

    grouped_edges = [
        (['a', 'b'], 'c'),
        ('c', ['d', 'e']),
    ]

    G = Graph()

    G.add_grouped_edges_from(grouped_edges)
    
    >>> print(G)
    Graph with 5 nodes and 4 edges

Set node objects
-----------------

Each node in the graph represents an execution, and the edges represent the data
flow. Therefore, we need to link the node to its execution method. In *mmodel*
a node object is a combination of:

1. callable, function, or function-like ("func")
2. callable return variable name ("output")
3. callable input parameter list ("inputs")
4. modifications list ("modifiers")

For linking the node object to the node, two methods are provided:
``set_node_object`` and ``set_node_objects_from``. 
The latter accepts a list of node objects. 

.. code-block:: python

    def add(x, y):
        """The sum of x and y."""
        return x + y


    def subtract(x, y):
        """The difference between x and y."""
        return x - y


    G.set_node_object(Node(node="a", func=add, output="z"))


    >>> print(G.node_metadata("a"))
    a

    add(x, y)
    return: z
    functype: function

    The sum of x and y.

    # or with multiple node objects
    # both nodes add input values but outputs in different parameter
    node_objects = [
        Node("a", add, output="z"),
        Node("b", subtract, output="m"),
    ]
    G.set_node_objects_from(node_objects)

    >>> print(G.node_metadata("b"))
    b

    subtract(x, y)
    return: m
    functype: function

    The difference between x and y.


The object is stored as a node attribute, and the function signature
(`inspect.Signature`) is stored. The parameter values are converted
to signature objects.
The note output is a single variable. If the node outputs multiple variables,
the return tuple is assigned to the defined output variable.

.. Note::

    To have the function docstring correctly displayed in the node's metadata,
    it needs to start with an upper case letter and end with a period.



Change function input parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To avoid re-defining functions using different input parameters or for functions
that only allow positional arguments (built-in functions and numpy.ufunc), the
"inputs" parameter of the ``set_node_object`` can change the node signature.
The signature replacement is a thin wrapper with a very small performance overhead.
The signature change only occurs at the node level. The original function is
not affected.

.. code-block:: python
    
    def add(a, b):
        """Sum of x and y."""
        return a + b

    G.set_node_object(Node("a", func=add, output="z", inputs=["m", "n"]))

    >>> print(G.node_metadata("a"))
    a

    add(m, n)
    return: z
    functype: function

    Sum of x and y.

.. Note:: 

    The graph variable flows are restricted to keyword arguments only for function parameters.
    They can be modified by changing the inputs of the function, and the modified
    function allows keyword arguments.

Built-in functions and functions without signature
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are different types of functions that ``inspect.signature`` cannot extract
the parameters from, namely:

1. python's built-in functions
2. *NumPy* ufuncs

mmodel can identify the above functions and replace the signature:

.. code-block:: python

    from operator import add

    G.set_node_object(Node("a", func=add, output="z", inputs=["m", "n"]))

    import numpy as np

    G.set_node_object(Node("b", func=np.sum, output="z", inputs=["m", "n"]))


    >>> print(G.get_node_object("a"))
    a

    add(m, n)
    return: z
    functype: builtin_function_or_method

    Same as a + b.


    >>> print(G.get_node_object("b"))
    b

    sum(m, n)
    return: d
    functype: function

    Sum of array elements over a given axis.

The ``set_node_object`` method can also accept additional keyword arguments that are
stored in the graph node attribute. The "doc" attribute is reserved for the docstring
of the function, however, it can be overridden by the user.

Function with variable length of arguments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In a *mmodel* graph, the argument length of a node is fixed. For a function with a variable
length of arguments, additional arguments can be provided using the input function.


.. code-block:: python

    def test_func_kwargs(a, b, **kwargs):
        return a + b, kwargs


    G.set_node_object(Node(node="a", func=test_func_kwargs, output="z", inputs=["m", "n", "p"]))

    >>> print(G.get_node_object("a"))
    a

    test_func_kwargs(m, n, p)
    return: z
    functype: function

    >>> G.get_node_object("a")(m=1, n=2, p=4)
    (3, {'p': 4})

Function with default arguments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For functions with default arguments, the inputs can be shorter than the total number
of parameters.

.. code-block:: python

    def test_func_defaults(m, n, p=2):
        return m + n + p
    
    G.set_node_object(Node("a", func=test_func_defaults, output="z", inputs=["m", "n"]))

    >>> print(G.get_node_object("a"))
    a

    test_func_defaults(m, n)
    return: z
    functype: function
    
    >>> G.get_node_object("a")(m=1, n=2)
    5

.. Note::

    To avoid performance overhead, signature_modifier modifies the signature in order.
    Currently, it is not possible to replace selected parameters.

Name and docstring
----------------------

The name and graph string behaves as the *networkx* graphs. To add the name to the graph:


.. code-block:: python
    
    # during graph definition
    G = Graph(name="Graph Example")

    # after definition
    # G.graph['name'] = 'ModelGraph Example'

    >>> print(G)
    Graph named 'Graph Example' with 0 nodes and 0 edges

Mutability
------------

The graph object is mutable. A shallow or deepcopy might be needed to create a copy
of the graph.

.. code-block:: python
    
    G.copy() # shallow copy
    G.deepcopy() # deep copy

For more ways to interact with ``Graph`` and ``networkx.graph`` see
:doc:`graph reference </ref_graph>`.
