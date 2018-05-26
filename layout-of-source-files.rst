********************************
솔리디티 소스파일 레이아웃
********************************

소스파일에는 지시문과 프라그마 지시문을 포함한 여러개의 컨트랙트를 정의할 수 있습니다.

.. index:: ! pragma, version

.. _version_pragma:

버전 프라그마
==============

소스파일은 '버전 프라그마'라고 불리는 버전 정보에 대한 주석을 달아야 하는데,
이는 최신 버전의 컴파일러가 호환되지 않는 방식의 컴파일을 막기 위함입니다.
우리는 그러한 변화를 절대적으로 최소화하려 노력하고 있으며, 특히 의미 변경이 구문의 변경을 요구하도록 변경 사항을 도입하지만 이것이 항상 가능하지는 않습니다.
그렇기 때문에 변경 사항이 포함된 릴리즈에 대해서는 ChangeLog를 읽어보는 것이 좋습니다. 그러한 릴리즈에는 항상 ``0.x.0`` 나 ``x.0.0`` 같은 버전 형식이 포함되어 있습니다.

버전 프라그마는 아래와 같이 사용됩니다.::

  pragma solidity ^0.4.0;

위와 같이 작성된 소스파일은 0.4.0 이전 버전의 컴파일러와 호환되지 않으며,
0.5.0 이상의 버전에서도 컴파일되지 않습니다. (두번째 조건의 버전정보는 ``^``을 사용하여 나타낸다). 이러한 방식은 ``0.5.0``버전까지 큰 변화가 없이 의도하고자 하는 방식으로 코드를 컴파일 할수 있게 합니다. 
우리는 

It is possible to specify much more complex rules for the compiler version,
the expression follows those used by `npm <https://docs.npmjs.com/misc/semver>`_.

.. index:: source file, ! import

.. _import:

Importing other Source Files
============================

Syntax and Semantics
--------------------

Solidity supports import statements that are very similar to those available in JavaScript
(from ES6 on), although Solidity does not know the concept of a "default export".

At a global level, you can use import statements of the following form:

::

  import "filename";

This statement imports all global symbols from "filename" (and symbols imported there) into the
current global scope (different than in ES6 but backwards-compatible for Solidity).

::

  import * as symbolName from "filename";

...creates a new global symbol ``symbolName`` whose members are all the global symbols from ``"filename"``.

::

  import {symbol1 as alias, symbol2} from "filename";

...creates new global symbols ``alias`` and ``symbol2`` which reference ``symbol1`` and ``symbol2`` from ``"filename"``, respectively.

Another syntax is not part of ES6, but probably convenient:

::

  import "filename" as symbolName;

which is equivalent to ``import * as symbolName from "filename";``.

Paths
-----

In the above, ``filename`` is always treated as a path with ``/`` as directory separator,
``.`` as the current and ``..`` as the parent directory.  When ``.`` or ``..`` is followed by a character except ``/``,
it is not considered as the current or the parent directory.
All path names are treated as absolute paths unless they start with the current ``.`` or the parent directory ``..``.

To import a file ``x`` from the same directory as the current file, use ``import "./x" as x;``.
If you use ``import "x" as x;`` instead, a different file could be referenced
(in a global "include directory").

It depends on the compiler (see below) how to actually resolve the paths.
In general, the directory hierarchy does not need to strictly map onto your local
filesystem, it can also map to resources discovered via e.g. ipfs, http or git.

Use in Actual Compilers
-----------------------

When the compiler is invoked, it is not only possible to specify how to
discover the first element of a path, but it is possible to specify path prefix
remappings so that e.g. ``github.com/ethereum/dapp-bin/library`` is remapped to
``/usr/local/dapp-bin/library`` and the compiler will read the files from there.
If multiple remappings can be applied, the one with the longest key is tried first. This
allows for a "fallback-remapping" with e.g. ``""`` maps to
``"/usr/local/include/solidity"``. Furthermore, these remappings can
depend on the context, which allows you to configure packages to
import e.g. different versions of a library of the same name.

**solc**:

For solc (the commandline compiler), these remappings are provided as
``context:prefix=target`` arguments, where both the ``context:`` and the
``=target`` parts are optional (where target defaults to prefix in that
case). All remapping values that are regular files are compiled (including
their dependencies). This mechanism is completely backwards-compatible (as long
as no filename contains = or :) and thus not a breaking change. All imports
in files in or below the directory ``context`` that import a file that
starts with ``prefix`` are redirected by replacing ``prefix`` by ``target``.

So as an example, if you clone
``github.com/ethereum/dapp-bin/`` locally to ``/usr/local/dapp-bin``, you can use
the following in your source file:

::

  import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;

and then run the compiler as

.. code-block:: bash

  solc github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ source.sol

As a more complex example, suppose you rely on some module that uses a
very old version of dapp-bin. That old version of dapp-bin is checked
out at ``/usr/local/dapp-bin_old``, then you can use

.. code-block:: bash

  solc module1:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ \
       module2:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin_old/ \
       source.sol

so that all imports in ``module2`` point to the old version but imports
in ``module1`` get the new version.

Note that solc only allows you to include files from certain directories:
They have to be in the directory (or subdirectory) of one of the explicitly
specified source files or in the directory (or subdirectory) of a remapping
target. If you want to allow direct absolute includes, just add the
remapping ``=/``.

If there are multiple remappings that lead to a valid file, the remapping
with the longest common prefix is chosen.

**Remix**:

`Remix <https://remix.ethereum.org/>`_
provides an automatic remapping for github and will also automatically retrieve
the file over the network:
You can import the iterable mapping by e.g.
``import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;``.

Other source code providers may be added in the future.


.. index:: ! comment, natspec

Comments
========

Single-line comments (``//``) and multi-line comments (``/*...*/``) are possible.

::

  // This is a single-line comment.

  /*
  This is a
  multi-line comment.
  */


Additionally, there is another type of comment called a natspec comment,
for which the documentation is not yet written. They are written with a
triple slash (``///``) or a double asterisk block(``/** ... */``) and
they should be used directly above function declarations or statements.
You can use `Doxygen <https://en.wikipedia.org/wiki/Doxygen>`_-style tags inside these comments to document
functions, annotate conditions for formal verification, and provide a
**confirmation text** which is shown to users when they attempt to invoke a
function.

In the following example we document the title of the contract, the explanation
for the two input parameters and two returned values.

::

    pragma solidity ^0.4.0;

    /** @title Shape calculator. */
    contract shapeCalculator {
        /** @dev Calculates a rectangle's surface and perimeter.
          * @param w Width of the rectangle.
          * @param h Height of the rectangle.
          * @return s The calculated surface.
          * @return p The calculated perimeter.
          */
        function rectangle(uint w, uint h) returns (uint s, uint p) {
            s = w * h;
            p = 2 * (w + h);
        }
    }
