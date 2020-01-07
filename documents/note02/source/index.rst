WISHBONE classic mode ambiguities
=================================

This document describes ambiguities in the classic WISHBONE mode and how
to fix it.

The problems
------------

The WISHBONE specification doesn't clearly defines when a transfer is
completed.

Let's start with a simple single read cycle (see
:numref:`singlereadcycle`).  This diagram is very similar to the ones
of the b3 specification like the single read cycle (figure 3.3)

.. _singlereadcycle:
.. wavedrom::
   :caption: SINGLE READ cycle.

   { "signal": [
       { "name": "CLK_I",  "wave": "P..." },
       { "name": "ADR_O()", "wave": "x=.x", "data": ["VALID"] },
       { "name": "DAT_I()", "wave": "x.=x", "data": ["VALID"] },
       { "name": "DAT_O()", "wave": "x..." },
       { "name": "WE_O", "wave": "x0.x" },
       { "name": "SEL_O()", "wave": "x=.x", "data": ["VALID"] },
       { "name": "STB_O", "wave": "01.0" },
       { "name": "CYC_O", "wave": "01.0" },
       { "name": "ACK_I", "wave": "0.10" }
    ],
    "config": { "hscale": 2 },
    "head": { "tick": -1 }
   }

Note that the [ACK_I] is sampled as asserted at the rising edge 2 of [CLK_I],
so [STB_O] and [CYC_O] are negated before the next rising edge.

But why is [ACK_I] negated after rising edge 2 ?  Remember:

**RULE 3.50**
  SLAVE interfaces MUST be designed so that the [ACK_O], [ERR_O], and [RTY_O]
  signals are asserted and negated in respone to the assertion and negation
  of [STB_I].

So according the this rule, it was negated because [STB_I] (which is the
slave side of [STB_O]) was negated.  But it is before the next clock
rising edge, so it was asynchronously negated.  So if the design was fully
synchronous, the diagram would be as in :numref:`synchreadcycle`.


.. _synchreadcycle:
.. wavedrom::
   :caption: SINGLE READ cycle with synchronous ACK.

   { "signal": [
       { "name": "CLK_I",  "wave": "P...." },
       { "name": "ADR_O()", "wave": "x=.x.", "data": ["VALID"] },
       { "name": "DAT_I()", "wave": "x.=x.", "data": ["VALID"] },
       { "name": "DAT_O()", "wave": "x...." },
       { "name": "WE_O", "wave": "x0.x." },
       { "name": "SEL_O()", "wave": "x=.x.", "data": ["VALID"] },
       { "name": "STB_O", "wave": "01.0." },
       { "name": "CYC_O", "wave": "01.0." },
       { "name": "ACK_I", "wave": "0.1.0" }
       ],
	  "config": { "hscale": 2 },
	  "head": { "tick": -1 }
	}

According to the "standard wishbone protocol":

  At every rising edge of [CLK_I], the terminating signal is sampled.  If
  it is asserted, then [STB_O] is negated.

Although this is not a rule, it means that [STB_O] must be negated for
rising edge 3.

When is the transfer completed ?  It would make sense to consider the
transfer completed at rising edge 3, when [STB_O] can be sampled as
negated.  It would also make sense not to consider the state of
[ACK_I] because according to rule 3.55 and permission 3.35 the slave
interface may hold [ACK_O] in the asserted state.  As a consequence, a
transfer needs at least two cycle: one with [STB_O] asserted and one
with [STB_O] negated.

The first ambiguity is between rule 3.55 and rule 3.50.  The former says
that [ACK_O] may be hold in the asserted state while the latter says it
must be asserted and negated in response of [STB_I].

The second ambiguity is in:

**observation 3.40**
  The asynchronous assertion of [ACK_O], [ERR_O], and [RTY_O] assures
  that the interface can accomplish one data transfer per clock
  cycle. [...]

Because of the standard wishbone protocol, [STB_O] must be negated before
being asserted.  So at most you can have one data transfer every 2 clock
cycles.

Solutions proposal
------------------

First note that this issue is not present with the pipelined mode added by
the b4 specification.

There are two ways to fix these ambiguities.  Either by slightly changing
the wording, or by slightly modifying the protocol.

Editorial changes
`````````````````

For the first ambiguity, it might be worth clarifying permission 3.35 and rule
3.55 first.  If they mean that it is possible to design a slave that always
assert [ACK_O], add the 'always' word in permission 3.35.  If they mean
something else (like holding [ACK_O] in the asserted state for many cycles),
the consequences have to be discussed.

Then the rule 3.55 could be remove because it doesn't give any direction on
how to operate.  But the fact that [ACK_I] may be hold in the asserted state
should be considered in rule 3.50.  Maybe adding "unless [ACK_O] is
always asserted" to rule 3.50.

For the second ambiguity, just stop saying that one data transfer per
clock cycle is possible using classic mode.

Changing the protocol
`````````````````````

The editorial changes don't fully clarify when a transaction finishes.  The
protocol could be improved if the transactions finish when [ACK_O] is
asserted.  And to avoid ambiguities, it must be asserted for one clock
cycle.  As a consequence, [STB_O] doesn't have to be negated before the
next data transfer as the transaction is finished.  Using this definition
allows one data transfer per cycle (which fixes the second ambiguity).

Is it incompatible with existing designs ?  First note that most (if not
all) diagrams use a single cycle [ACK_O].  Second, also note that for
asynchrounous SLAVE (which is almost implied by the wishbone specification),
this doesn't change anything.  Finally, this is also the behaviour of
the pipelined mode protocol.
