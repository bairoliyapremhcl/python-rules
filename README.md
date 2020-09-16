## Rules vs. Code
For most transaction-oriented database applications, backend database logic
is a substantial portion of the effort.
It includes _multi-table_ derivation and constraint logic,
and actions such as sending mail or messages.

Such backend logic is typically coded in `before_flush` events,
or database triggers and/or stored procedures.
The prevailing assumption is that such *domain-specific logic must surely be 
domain-specfic code.*  

The problem is that this is a lot of code.  Often nearly half the
effort for a transactional database-oriented systems,
it is time-consuming, complex and error-prone.

This project introduces a _declarative
alternative_: you specify a set of **_spreadsheet-like
rules,_** which are then executed by a login engine operating
as a plugin to sqlalchemy.

#### Rules: 40X more concise, automatic optimization and re-use

This declarative, *rule-oriented* approach contrasts sharply with
traditional hand-coded *procedural* `after_flush` events or triggers:

| Consideration |      Declarative Rules    | Procedural (`after_flush`, Triggers, ...) |
| ------------- | ------------- | --------- |
| **Conciseness**  | **5 spreadsheet-like rules** implement the check-credit requirement (shown below) | The same logic requires **200 hundred of lines** of code [(shown here)](https://github.com/valhuber/python-rules/wiki/by-code) - a factor of 40:1|
| **Performance** | SQLs are *automatically pruned and minimized* (example below)| Optimizations require hand-code, often over-looked due to project time pressure |
| **Quality** | Rules are *automatically re-used* over all transactions, minimizing missed corner-cases| Considerable test and debug is required to find and address all corner cases, with high risk of bugs |
| **Agility** | Rule execution is *automatically re-ordered* per dependencies, simplifying iteration cycles<br><br>Business Users can read the rules, and collaborate<br><br>Collaboration is further supported by running screens - see also Fab-QuickStart below | Changes require code to be re-engineered, at substantial cost and time |

This can represent a meaningful reduction in project delivery.
Experience has shown that such rules can address *over 95%* of
the backend logic, reducing such logic by **40X** (200 vs. 5).

Importantly, logic is
* *Extensible:* Rules are complemented by Python events,
so you can address the last 5%
* *Manageable:* logic is expressed in Python, enabling the use of
standard IDE and Source Code Control systems
* *Debuggable:* Debug your logic with logs that show which rules execute,
and breakpoints in formula/constraint/action rules
expressed in Python

## Overview
The subject database is an adaption of the Northwind database,
with a few rollup columns added.
For those not familiar, this is basically
Customers, Orders, OrderDetails and Products.

### Architecture
<img src="https://github.com/valhuber/python-rules/blob/master/images/architecture.png" width="500">

1. Your logic is **declared** as Python functions (see example below).

1. Your application makes calls on `sqlalchemy` for inserts, updates and deletes.
This code can be hand-written, or via generators such as Flask AppBuilder.

1. The **python-rules** logic engine handles sqlalchemy `before_flush` events on
`Mapped Tables`

1. The logic engine operates much like a spreadsheet:
**watch** for changes at the attribute level,
**react** by running rules that referenced changed attributes,
which can **chain** to still other attributes that refer to
_those_ changes.  Note these might be in different tables,
providing automation for _multi-table logic_.

Logic does not apply to updates outside sqlalchemy,
or to sqlalchemy batch updates or unmapped sql updates.


## Declaring Logic as Spreadsheet-like Rules
Logic is declared as spreadsheet-like rules as shown below
from  [`nw_rules_bank.py`](nw/nw_logic/nw_rules_bank.py),
activated in [`__init__.py`](nw/nw_logic/__init__.py).
The logic below implements the *check credit* requirement:
* *the balance must not exceed the credit limit,*
* *where the balance is the sum of the unshipped order totals*
* *which is the rollup of OrderDetail Price * Quantities:*
```python
Rule.constraint(validate="Customer", as_condition="row.Balance <= row.CreditLimit",
                error_msg="balance ({row.Balance}) exceeds credit ({row.CreditLimit})")
Rule.sum(derive="Customer.Balance", as_sum_of="OrderList.AmountTotal", where="row.ShippedDate is None")

Rule.sum(derive="Order.AmountTotal", as_sum_of="OrderDetailList.Amount")

Rule.formula(derive="OrderDetail.Amount", as_exp="row.UnitPrice * row.Quantity")
Rule.copy(derive="OrderDetail.UnitPrice", from_parent="ProductOrdered.UnitPrice")
Rule.formula(derive="OrderDetail.ShippedDate", as_exp="row.OrderHeader.ShippedDate")

Rule.sum(derive="Product.UnitsShipped", as_sum_of="OrderList.Quantity",
         where="row.ShippedDate is not None")
Rule.formula(derive="Product.UnitsInStock", calling=units_shipped)

```
The specification is fully executable, and governs around a
dozen transactions.  Here we look at 2 simple examples:

* **Add Order (Check Credit) -** enter an order/orderdetails,
and rollup to AmountTotal / Balance to check CreditLimit

* **Ship / Unship an Order (Adjust Balance) -** when an Order's `DateShippped`
is changed, adjust the Customers `Balance`

These representatively complex transactions illustrate common logic execution patterns:


## Logic Execution: Watch, React, Chain
The engine operates much as you might imagine a spreadsheet:

* **Watch** - changes are detected at the *attribute* level

* **React** - derivation rules referencing changes are (re)executed
(forward chaining *rule inference*); unreferenced rules are pruned.

* **Chain** - if recomputed values are referenced by still other rules,
*these* are re-executed.  Note this can be in other tables, thus
automating multi-table transaction logic.

   * Unlike coarse-grained triggers or event handlers at the table level,
   derivations are fine-grained at the attribute level.
   * This enables the rules system to automate efficiencies like pruning
   and adjustment, as described below
   
   
#### Example: Add Order - Multi-Table Adjustment, Chaining
The **Add Order** example illustrates chaining:

* OrderDetails are referenced by the Orders' `AmountTotal` sum rule,
so `AmountTotal` is adjusted

* The `AmountTotal` is referenced by the Customers' `Balance`,
so it is adjusted

* And the Credit Limit constraint is checked 
(exceptions are raised if constraints are violated)

All of the dependency management to see which attribute have changed,
logic ordering, the sql commands to read and adjust rows, and the chaining
are fully automated by the engine, based on the rules above.
This is how 5 rules represent the same logic as 200 lines of code.

Key points:

##### Multi-Table Logic
The `sum` rule "watching" `OrderDetail.AmountTotal` changes is in
a different table: `Orders`.  So, the "react" logic has to
perform a multi-table transaction.

##### Optimizations: Adjustment (vs. nested `sum` queries)

Note that rules declare _end conditions_, enabling / _obligating_
the engine to optimize execution (like a sql query optimizer).
For example, sum/count aggregate processing is
_not_ processed as an expensive aggregate query,
but rather as an *1 row adjustment*.

Here, computing the
balance does _not_ result in a sum over all the Orders, and OrderDetails.

  * Contrast this to approaches in other systems where
the balance is recomputed with expensive aggregate queries over *all*
the customers' orders, and *all* their OrderDetails.

  *   Imagine, for example, a customer might have
   thousands of Orders, each with thousands of OrderDetails.


[See here](../../wiki/Multi-Table-Logic-Execution)
for more information on Rule Execution.


#### Example: Ship Order - Pruning, Adjustment and Cascade
The **ship / unship order** example illustrates pruning and adjustment:

* if `DueDate` is altered, nothing is dependent on that,
so the rule is **pruned** from the logic execution.  The result
is a 1 row transaction - no SQL overhead from rules.

* if `ShippedDate` altered, 2 kinds of multi-table logic are triggered:

   * the logic engine **adjusts** the `Customer.Balance` with a 1 row update.
   
       * Note the here, the triggering event is a change to the `where` condition,
   rather than a change to the summed value.
   
   * the `ShippedDate` is _also_ referenced by the `OrderDetail.ShippedDate` rule,
   so the system **cascades** the change to each `OrderDetail`
   to reevaluate referring rules.
   
       * This **further _chains_** to _adjust_ `Product.UnitsInStock`,
       whose change recomputes `Product.UnitsInStock` (see below)
 

#### State Transition Logic (old values)
Logic often depends on the old and new state of a row.
For example, here is the function used to compute `Product.UnitsInStock`:
```python
def units_shipped(row: Product, old_row: Product, logic_row: LogicRow):
    result = row.UnitsInStock - (row.UnitsShipped - old_row.UnitsShipped)
    return result
```
Note this logic is in Python: you can invoke Python functions,
set breakpoints, etc.

#### DB-generated Keys
DB-generated keys are often tricky (how do you insert
items if you don't know the db-generated orderId?), shown here in `Order`
and `OrderDetail`.  These were well-handled by sqlalchemy,
where adding OrderDetail rows into the Orders' collection automatically
set the foreign keys.

## Installation
Relies on `from __future__ import annotations`, so requires Python 3.7.

Using your IDE (works with PyCharm, VSC fails on imports) or command line: 
```
git clone
cd python-rules
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
```
The project includes:
* the logic engine that executes the rules
* the sample database (sqlite, so no db install is required)
* business logic, both
[by-code](https://github.com/valhuber/python-rules/wiki/by-code) and
[by-rules,](https://github.com/valhuber/python-rules/wiki/by-rules)
to facilitate comparison
   * control whether logic is via rules or code by altering`by_rules` in
   [`__init__.py`](https://github.com/valhuber/python-rules/blob/master/nw/nw_logic/__init__.py)
* a test folder that runs various sample transactions

You can run the programs in the `nw/trans-tests` folder,
and/or review this readme and the wiki.

## Status: Running, Under Development
Essential functions running on 9/6/2020: able to save order (a multi-table transaction - certain paths of copy, formula, constraint and sum rules).  Not complete, under active development.


## Flask App Builder
You can also run an app (generated by [fab-quick-start](https://github.com/valhuber/fab-quick-start/wiki))
to exolore the `nw` database, though this is not currently enforcing logic.

Fab-quick-start builds default web apps in just a few minutes.  Such working software can support agile business / IT collaboration.

```
cd nw-app
export FLASK_APP=app
flask run
```
