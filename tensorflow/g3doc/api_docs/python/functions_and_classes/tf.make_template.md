### `tf.make_template(name_, func_, **kwargs)` {#make_template}

Given an arbitrary function, wrap it so that it does variable sharing.

This wraps `func_` in a Template and partially evaluates it. Templates are
functions that create variables the first time they are called and reuse them
thereafter. In order for `func_` to be compatible with a `Template` it must
have the following properties:

* The function should create all trainable variables and any variables that
   should be reused by calling `tf.get_variable`. If a trainable variable is
   created using `tf.Variable`, then a ValueError will be thrown. Variables
   that are intended to be locals can be created by specifying
   `tf.Variable(..., trainable=false)`.
* The function may use variable scopes and other templates internally to
    create and reuse variables, but it shouldn't use `tf.get_variables` to
    capture variables that are defined outside of the scope of the function.
* Internal scopes and variable names should not depend on any arguments that
    are not supplied to `make_template`. In general you will get a ValueError
    telling you that you are trying to reuse a variable that doesn't exist
    if you make a mistake.

In the following example, both `z` and `w` will be scaled by the same `y`. It
is important to note that if we didn't assign `scalar_name` and used a
different name for z and w that a `ValueError` would be thrown because it
couldn't reuse the variable.

```python
def my_op(x, scalar_name):
  var1 = tf.get_variable(scalar_name,
                         shape=[],
                         initializer=tf.constant_initializer(1))
  return x * var1

scale_by_y = tf.make_template('scale_by_y', my_op, scalar_name='y')

z = scale_by_y(input1)
w = scale_by_y(input2)
```

As a safe-guard, the returned function will raise a `ValueError` after the
first call if trainable variables are created by calling `tf.Variable`.

If all of these are true, then 2 properties are enforced by the template:

1. Calling the same template multiple times will share all non-local
    variables.
2. Two different templates are guaranteed to be unique, unless you reenter the
    same variable scope as the initial definition of a template and redefine
    it. An examples of this exception:

```python
def my_op(x, scalar_name):
  var1 = tf.get_variable(scalar_name,
                         shape=[],
                         initializer=tf.constant_initializer(1))
  return x * var1

with tf.variable_scope('scope') as vs:
  scale_by_y = tf.make_template('scale_by_y', my_op, scalar_name='y')
  z = scale_by_y(input1)
  w = scale_by_y(input2)

# Creates a template that reuses the variables above.
with tf.variable_scope(vs, reuse=True):
  scale_by_y2 = tf.make_template('scale_by_y', my_op, scalar_name='y')
  z2 = scale_by_y2(input1)
  w2 = scale_by_y2(input2)
```

Note: The full variable scope is captured at the time of the first call.

Note: `name_` and `func_` have a following underscore to reduce the likelihood
of collisions with kwargs.

##### Args:


*  <b>`name_`</b>: A name for the scope created by this template. If necessary, the name
    will be made unique by appending `_N` to the name.
*  <b>`func_`</b>: The function to wrap.
*  <b>`**kwargs`</b>: Keyword arguments to apply to `func_`.

##### Returns:

  A function that will enter a `variable_scope` before calling `func_`. The
  first time it is called, it will create a non-reusing scope so that the
  variables will be unique.  On each subsequent call, it will reuse those
  variables.

##### Raises:


*  <b>`ValueError`</b>: if the name is None.

