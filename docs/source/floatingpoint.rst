.. _tut-fp-issues:

**************************************************
浮点数算法：争议和限制
**************************************************

.. sectionauthor:: Tim Peters <tim_one@users.sourceforge.net>


浮点数在计算机中表达为二进制(binary)小数。例如：十进制小数::

   0.125

是 1/10 + 2/100 + 5/1000 的值，同样二进制小数::

   0.001

是 0/2 + 0/4 + 1/8。这两个数值相同。唯一的实质区别是第一个写为十进制小数记法，第二个是二进制。 

遗憾的是，大多数十进制小数不能精确的表达二进制小数。 

这个问题更早的时候首先在十进制中发现。考虑小数形式的 1/3，你可以来个十进制的近似值::

   0.3

或者更进一步的::

   0.33

或者更进一步的::

   0.333

诸如此类。如果你写多少位，这个结果永远不是精确的 1/3，但是可以无限接近 1/3。 

同样，无论在二进制中写多少位，十进制数 0.1 都不能精确表达为二进制小数。二进制来表达 1/10 是一个无限循环小数::

   0.0001100110011001100110011001100110011001100110011...

在任意无限位数值中中止，你可以得到一个近似值。

在一个典型的机器上运行 Python，一共有 53 位的精度来表示一个浮点数，所以当你输入十进制的 ``0.1`` 的时候，看到是一个二进制的小数::

   0.00011001100110011001100110011001100110011001100110011010

非常接近，但是不完全等于, 1/10.

这是很容易忘记，存储的值是一个近似的原小数，由于浮体的方式，显示在提示符的解释。Python 中只打印一个小数近似的真实机器所存储的二进制近似的十进制值。如果 Python 
要打印存储的二进制近似真实的十进制值 0.1，那就要显示::

   >>> 0.1
   0.1000000000000000055511151231257827021181583404541015625

这么多位的数字对大多数人是没有用的，所以 Python 显示一个舍入的值::

   >>> 0.1
   0.1

认识到这个幻觉的真相很重要：机器不能精确表达 1/10，你可以简单的截断显示真正的机器值。这里还有另一个惊奇之处。例如，下面::

   >>> 0.1 + 0.2
   0.30000000000000004

需要注意的是这在二进制浮点数是非常自然的：它不是 Python 的 bug，也不是你的代码的 bug。你会看到只要你的硬件支持浮点数算法，所有的语言都会有这个现象(尽管有些语言可能默认或完全不 *显示* 这个差异)。

还有其它意想不到的。例如，如果你舍入2.675到两位小数，你得到的是::

   >>> round(2.675, 2)
   2.67

内置 `round() <https://docs.python.org/2.7/library/functions.html#round>`_ 函数的文档说它舍入到最接近的值。因为小数 2.675 正好是 2.67 和 2.68 的中间，你可能期望这里的结果是（二进制近似为) 2.68。但是不是的，因为当十进制字符串 2.675 转换为一个二进制浮点数时，它仍然被替换为一个二进制的近似值，其确切的值是::

   2.67499999999999982236431605997495353221893310546875

因为这个近似值稍微接近 2.67 而不是 2.68，所以向下舍入。

如果你的情况需要考虑十进制的中位数是如何被舍入的，你应该考虑使用 `decimal <https://docs.python.org/2.7/library/decimal.html#module-decimal>`_ 模块。顺便说一下，`decimal <https://docs.python.org/2.7/library/decimal.html#module-decimal>`_ 模块还提供了很好的方式可以“看到”任何 Python 浮点数的精确值::

   >>> from decimal import Decimal
   >>> Decimal(2.675)
   Decimal('2.67499999999999982236431605997495353221893310546875')

这个问题在于存储 “0.1” 的浮点值已经达到 1/10 的最佳精度了，所以尝试截断它不能改善：它已经尽可能的好了。另一个影响是因为 0.1 不能精确的表达 1/10，对 10 个 0.1 的值求和不能精确的得到 1.0，即::

   >>> sum = 0.0
   >>> for i in range(10):
   ...     sum += 0.1
   ...
   >>> sum
   0.9999999999999999

浮点数据算法产生了很多诸如此类的怪异现象。在 “表现错误” 一节中，这个 “0.1” 问题详细表达了精度问题。更完整的其它常见的怪异现象请参见 `浮点数危害 <http://www.lahey.com/float.htm>`_ 。 最后我要说，“没有简单的答案”。还是不要过度的敌视浮点数！

Python 浮点数操作的错误来自于浮点数硬件，大多数机器上同类的问题每次计算误差不超过 2\*\*53 分之一。对于大多数任务这已经足够让人满意了。但是你要在心中记住这不是十进制算法，每个浮点数计算可能会带来一个新的精度错误。 

问题已经存在了，对于大多数偶发的浮点数错误，你应该比对最终显示结果是否符合你的期待。`str() <https://docs.python.org/2.7/library/functions.html#str>`_ 通常够用了，完全的控制参见 `字符串格式化 <https://docs.python.org/2.7/library/string.html#formatstrings>`_ 中 `str.format <https://docs.python.org/2.7/library/stdtypes.html#str.format>`_ 方法的格式化方式。


.. _tut-fp-error:

表达错误
====================

这一节详细说明 “0.1” 示例，教你怎样自己去精确地分析此类案例。假设这里你已经对浮点数表示有基本的了解。 

:dfn:`Representation error` 提及事实上有些(实际是大多数)十进制小数不能精确的表示为二进制小数。这是 Python (或 Perl，C，C++，Java，Fortran 以及其它很多)语言往往不能按你期待的样子显示十进制数值的根本原因::

   >>> 0.1 + 0.2
   0.30000000000000004

这 是为什么？1/10 不能精确的表示为二进制小数。大多数今天的机器(2000 年十一月)使用 IEEE-754 浮点数算法，大多数平台上 Python 将浮点数映射为 IEEE-754 “双精度浮点数”。754 双精度包含 53 位精度，所以计算机努力将输入的 0.1 转为 J/2**N 最接近的二进制小数。*J* 是一个 53 位的整数。改写::

   1 / 10 ~= J / (2**N)

为::

   J ~= 2**N / 10

*J* 重现时正是 53 位(是 ``>= 2**52`` 而非 ``< 2**53``)，*N* 的最佳值是 56::

   >>> 2**52
   4503599627370496
   >>> 2**53
   9007199254740992
   >>> 2**56/10
   7205759403792793

因此，56 是保持 J 精度的唯一 *N* 值。*J* 最好的近似值是整除的商::

   >>> q, r = divmod(2**56, 10)
   >>> r
   6

因为余数大于 10 的一半，最好的近似是取上界::

   >>> q+1
   7205759403792794

因此在 754 双精度中 1/10 最好的近似值是是 2\*\*56，或::

   7205759403792794 / 72057594037927936

要注意因为我们向上舍入，它其实比 1/10 稍大一点点。如果我们没有向上舍入，它会比 1/10 稍小一点。但是没办法让它恰好是 1/10！ 

所以计算机永远也不 “知道” 1/10：它遇到上面这个小数，给出它所能得到的最佳的 754 双精度实数::

   >>> .1 * 2**56
   7205759403792794.0

如果我们用 10**30 除这个小数，会看到它最大30位(截断后的)的十进制值::

   >>> 7205759403792794 * 10**30 // 2**56
   100000000000000005551115123125L

这表示存储在计算机中的实际值近似等于十进制值 0.100000000000000005551115123125。Python 显示时取 17 位精度为 0。10000000000000001(是的，在任何符合 754 的平台上，都会由其C库转换为这个最佳近似——你的可能不一样！)。