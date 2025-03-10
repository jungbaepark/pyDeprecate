# pyDeprecate

**Simple tooling for marking deprecated functions or classes and re-routing to the new successors' instance.**

[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/pyDeprecate)](https://pypi.org/project/pyDeprecate/)
[![PyPI Status](https://badge.fury.io/py/pyDeprecate.svg)](https://badge.fury.io/py/pyDeprecate)
[![PyPI Status](https://pepy.tech/badge/pyDeprecate)](https://pepy.tech/project/pyDeprecate)
[![Conda](https://img.shields.io/conda/v/conda-forge/pyDeprecate?label=conda&color=success)](https://anaconda.org/conda-forge/pyDeprecate)
![Conda](https://img.shields.io/conda/dn/conda-forge/pyDeprecate)
[![license](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/Borda/pyDeprecate/blob/master/LICENSE)

[![CI testing](https://github.com/Borda/pyDeprecate/actions/workflows/ci_testing.yml/badge.svg?branch=main&event=push)](https://github.com/Borda/pyDeprecate/actions/workflows/ci_testing.yml)
[![Code formatting](https://github.com/Borda/pyDeprecate/actions/workflows/code-format.yml/badge.svg?branch=main&event=push)](https://github.com/Borda/pyDeprecate/actions/workflows/code-format.yml)
[![codecov](https://codecov.io/gh/Borda/pyDeprecate/branch/main/graph/badge.svg?token=BG7RQ86UJA)](https://codecov.io/gh/Borda/pyDeprecate)
[![CodeFactor](https://www.codefactor.io/repository/github/borda/pydeprecate/badge)](https://www.codefactor.io/repository/github/borda/pydeprecate)
[![pre-commit.ci status](https://results.pre-commit.ci/badge/github/Borda/pyDeprecate/main.svg)](https://results.pre-commit.ci/latest/github/Borda/pyDeprecate/main)

<!--
[![Language grade: Python](https://img.shields.io/lgtm/grade/python/g/Borda/pyDeprecate.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/Borda/pyDeprecate/context:python)
-->

---

The common use-case is moving your functions across codebase or outsourcing some functionalities to new packages.
For most of these cases, you want to hold some compatibility, so you cannot simply remove past function, and also for some time you want to warn users that functionality they have been using is moved and not it is deprecated in favor of another function (which shall be used instead) and soon it will be removed completely.

Another good aspect is to do not overwhelm a user with too many warnings, so per function/class, this warning is raised only N times in the preferable stream (warning, logger, etc.).

## Installation

Simple installation from PyPI:
```bash
pip install pyDeprecate
```

<details>
  <summary>Other installations</summary>

  Simply install with pip from source:
  ```bash
  pip install https://github.com/Borda/pyDeprecate/archive/main.zip
  ```

</details>

## Use-cases

The functionality is kept simple and all default shall be reasonable, but still you can do extra customization such as:

* define user warning message and preferable stream
* extended argument mapping to target function/method
* define deprecation logic for self arguments
* specify warning count per:
    - called function (for func deprecation)
    - used arguments (for argument deprecation)
* define conditional skip (e.g. depending on some package version)

In particular the target values (cases):

- _None_ - raise only warning message (ignore all argument mapping)
- _True_ - deprecation some argument of itself (argument mapping shall be specified)
- _Callable_ - forward call to new methods (optional also argument mapping or extras)

### Simple function forwarding

It is very straight forward, you forward your function call to new function and all arguments are mapped:

```python
def base_sum(a: int = 0, b: int = 3) -> int:
    """My new function anywhere in codebase or even other package."""
    return a + b

# ---------------------------

from deprecate import deprecated

@deprecated(target=base_sum, deprecated_in="0.1", remove_in="0.5")
def depr_sum(a: int, b: int = 5) -> int:
    """
    My deprecated function which now has empty body
     as all calls are routed to the new function.
    """
    pass  # or you can just place docstring as one above

# call this function will raise deprecation warning:
#   The `depr_sum` was deprecated since v0.1 in favor of `__main__.base_sum`.
#   It will be removed in v0.5.
print(depr_sum(1, 2))
```
<details>
  <summary>sample output:</summary>
  ```
  3
  ```
</details>

### Advanced target argument mapping

Another more complex example is using argument mapping is:


<details>
  <summary>Advanced example</summary>

  ```python
  import logging
  from sklearn.metrics import accuracy_score
  from deprecate import deprecated, void

  @deprecated(
    # use standard sklearn accuracy implementation
    target=accuracy_score,
    # custom warning stream
    stream=logging.warning,
    # number or warnings per lifetime (with -1 for always_
    num_warns=5,
    # custom message template
    template_mgs="`%(source_name)s` was deprecated, use `%(target_path)s`",
    # as target args are different, define mapping from source to target func
    args_mapping={'preds': 'y_pred', 'target': 'y_true', 'blabla': None}
  )
  def depr_accuracy(preds: list, target: list, blabla: float) -> float:
      """My deprecated function which is mapping to sklearn accuracy."""
      # to stop complain your IDE about unused argument you can use void/empty function
      return void(preds, target, blabla)

  # call this function will raise deprecation warning:
  #   WARNING:root:`depr_accuracy` was deprecated, use `sklearn.metrics.accuracy_score`
  print(depr_accuracy([1, 0, 1, 2], [0, 1, 1, 2], 1.23))
  ```
  sample output:
  ```
  0.5
  ```

</details>


### Deprecation warning only

Base use-case with no forwarding and just raising warning :

```python
from deprecate import deprecated

@deprecated(target=None, deprecated_in="0.1", remove_in="0.5")
def my_sum(a: int, b: int = 5) -> int:
    """My deprecated function which still has to have implementation."""
    return a + b

# call this function will raise deprecation warning:
#   The `my_sum` was deprecated since v0.1. It will be removed in v0.5.
print(my_sum(1, 2))
```
<details>
  <summary>sample output:</summary>
  ```
  3
  ```
</details>

### Self argument mapping

We also support deprecation and argument mapping for the function itself:

```python
from deprecate import deprecated

@deprecated(
  # define as depreaction some self argument - mapping
  target=True, args_mapping={'coef': 'new_coef'},
  # common version info
  deprecated_in="0.2", remove_in="0.4",
)
def any_pow(base: float, coef: float = 0, new_coef: float = 0) -> float:
    """My function with deprecated argument `coef` mapped to `new_coef`."""
    return base ** new_coef

# call this function will raise deprecation warning:
#   The `any_pow` uses deprecated arguments: `coef` -> `new_coef`.
#   They were deprecated since v0.2 and will be removed in v0.4.
print(any_pow(2, 3))
```
<details>
  <summary>sample output:</summary>
  ```
  8
  ```
</details>

### Multiple deprecation levels

Eventually you can set multiple deprecation levels via chaining deprecation arguments as each could be deprecated in another version:

<details>
  <summary>Multiple deprecation levels</summary>

  ```python
  from deprecate import deprecated

  @deprecated(
    True, "0.3", "0.6", args_mapping=dict(c1='nc1'),
    template_mgs="Depr: v%(deprecated_in)s rm v%(remove_in)s for args: %(argument_map)s."
  )
  @deprecated(
    True, "0.4", "0.7", args_mapping=dict(nc1='nc2'),
    template_mgs="Depr: v%(deprecated_in)s rm v%(remove_in)s for args: %(argument_map)s."
  )
  def any_pow(base, c1: float = 0, nc1: float = 0, nc2: float = 2) -> float:
      return base**nc2

  # call this function will raise deprecation warning:
  #   DeprecationWarning('Depr: v0.3 rm v0.6 for args: `c1` -> `nc1`.')
  #   DeprecationWarning('Depr: v0.4 rm v0.7 for args: `nc1` -> `nc2`.')
  print(any_pow(2, 3))
  ```
  sample output:
  ```
  8
  ```

</details>

### Conditional skip

Conditional skip of which can be used for mapping between different target functions depending on additional input such as package version

```python
from deprecate import deprecated

FAKE_VERSION = 1

def version_greater_1():
    return FAKE_VERSION > 1

@deprecated(
  True, "0.3", "0.6", args_mapping=dict(c1='nc1'), skip_if=version_greater_1
)
def skip_pow(base, c1: float = 1, nc1: float = 1) -> float:
    return base**(c1 - nc1)

# call this function will raise deprecation warning
print(skip_pow(2, 3))

# change the fake versions
FAKE_VERSION = 2

# Will not raise any warning
print(skip_pow(2, 3))
```
<details>
  <summary>sample output:</summary>
  ```
  0.25
  4
  ```
</details>

This can be beneficial with multiple deprecation levels shown above...

### Class deprecation

This case can be quite complex as you may deprecate just some methods, here we show full class deprecation:

```python
class NewCls:
    """My new class anywhere in the codebase or other package."""

    def __init__(self, c: float, d: str = "abc"):
        self.my_c = c
        self.my_d = d

# ---------------------------

from deprecate import deprecated, void

class PastCls(NewCls):
    """
    The deprecated class shall be inherited from the successor class
     to hold all methods.
    """

    @deprecated(target=NewCls, deprecated_in="0.2", remove_in="0.4")
    def __init__(self, c: int, d: str = "efg"):
        """
        You place the decorator around __init__ as you want
         to warn user just at the time of creating object.
        """
        return void(c, d)

# call this function will raise deprecation warning:
#   The `PastCls` was deprecated since v0.2 in favor of `__main__.NewCls`.
#   It will be removed in v0.4.
inst = PastCls(7)
print(inst.my_c)  # returns: 7
print(inst.my_d)  # returns: "efg"
```
<details>
  <summary>sample output:</summary>
  ```
  7
  efg
  ```
</details>

## Contribution

Have you faced this in past or even now, do you have good ideas for improvement, all is welcome!
