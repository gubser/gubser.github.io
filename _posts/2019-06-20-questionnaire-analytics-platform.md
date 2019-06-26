---
title:  "From spreadsheets to a data-driven questionnaire and analytics platform"
comments: true
---
# From spreadsheets to a data-driven questionnaire and analytics platform

Spreadsheet applications (Microsoft Excel, LibreOffice Calc) are very powerful tools for dealing with data. They feature pivot tables for exploration, solvers for optimization problems and connectors to pull data from any database. Needless to say that most data analytics today is done in a spreadsheet application.

Our client, a renowned research institute, collects data from many third-party businesses and analyze their environmental impact. Up to now, the institute sends a spreadsheet questionnaire to each business by mail. Each has to fill in the values and send it back. Afterwards, the analysis is done in a series of spreadsheets as well.

When scaling up this approach, the researchers faced problems with data collection (missing/invalid values, back and forth emails for clarification) and in-house collaboration (heterogenous schema makes sharing data between projects difficult, also consolidation of analysis logic from many spreadsheets into a single one is an elaborate, error-prone task).

## Separation of data and logic is the key to success

To enable data reuse between projects, a global catalog of variables is introduced. The variable definition includes the data type (string, number, date, ..), the physical unit and minimum / maximum values (numbers only). If necessary, new variables can be defined quickly.

The analysis is no longer placed into spreadsheets but is written in Python code stored in Git repositories. These so called *analysis modules* enable code reuse between projects. A project can utilize multiple analysis modules. This allows them to build a versatile toolbox of modules (i.e. data preparation, calculation, allocation to key performance values)

The questionnaires are now interactive websites. Generated from the set of variables that are required by the project. This set is determined by analyzing the variables used in the chosen Python scripts.

The app consists of a frontend, written in Angular, and a REST-style API backend, written in C#. In addition we operate an ElasticSearch instance to provide fuzzy search and we implemented an endpoint to fetch data for ad-hoc analysis in [Jupyter Notebooks](https://jupyter.org/). What you cannot see in the following diagram is that the backend is compartmentalized further into *API* and *analytics engine*. There is no communication between them except via the database. This not only reduces the spaghetti force but it also allows us to scale the analytics engine horizontally if needed.

![A diagram of the high level components frontend, python subsystem, core backend, elasticsearch and the database](/assets/2019-06-20/ComponentOverview.svg)

In the remaining blog post, I'll focus on dependency analysis and execution of modules. [Let me know](https://twitter.com/eliogubser) if you are interested in another blog post about other parts of the system.

## Dependency analysis and execution schedule
Consider a project that utilizes the four modules A, B, C and D as seen in the diagram (way) below. Our goal is to know two things:

<style type="text/css">
    ol.alphabetic { list-style-type: lower-alpha; }
</style>

<ol class="alphabetic">
  <li>The set of variables that we need to add to the questionnaire.</li>
  <li>A feasible execution schedule that satisfies the input and output dependencies of the modules.</li>
</ol>

The whole process is summarized in this diagram and explained in the following sections:
![From Python code to a schedule](/assets/2019-06-20/FromModuleToSchedule.svg)

### Determine dependencies of a single module
Let's start by detecting the input and output variables in the Python code. An analysis module has to follow the rule that inputs are attributes on an object called `inp` and outputs are attributes on an object called `out`. Example:
```python
out.p = inp.x + inp.y
out.q = inp.y + 1
```

Because the modules are stored in Git repositories, we need to `git clone` each module and walk Python's [abstract syntax tree](https://docs.python.org/3/library/ast.html) of the module code. For example using this:

```python
import ast
import json
import sys

class DependencyFinder(ast.NodeVisitor):
    inputs = set()
    outputs = set()

    def visit_Attribute(self, node):
        if type(node.value) is ast.Name:
            if node.value.id == 'inp':
                self.inputs.add(node.attr)
            elif node.value.id == 'out':
                self.outputs.add(node.attr)
        self.generic_visit(node)

if len(sys.argv) > 1:
    finder = DependencyFinder()

    with open(sys.argv[1], 'r', encoding='utf-8') as f:
        code = f.read()
        tree = ast.parse(code)
        finder.visit(tree)

    print("inputs are", ', '.join(sorted(finder.inputs)))
    print("outputs are", ', '.join(sorted(finder.outputs)))
else:
print(f"usage: {sys.argv[0]} <python-file>")
```

Given the module code
```python
out.p = inp.x + inp.y
out.q = inp.y + 1
```
it outputs 
```bash
$ python3 analyze.py module.py
inputs are x, y
outputs are p, q
```

### Building the dependency graph

Then, knowing all inputs and outputs, we can build a dependency graph and find the answers to questions above:
<ol class="alphabetic">
  <li>All input variables of a module that aren't output variables of some other module need to go into the questionnaire.</li>
  <li>By walking the whole tree starting from the `finish` node, we can determine the order of execution (= a feasible schedule).</li>
</ol>

