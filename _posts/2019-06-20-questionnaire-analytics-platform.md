---
title:  "From spreadsheets to a data-driven questionnaire and analytics platform"
comments: true
---
# From spreadsheets to a data-driven questionnaire and analytics platform

Spreadsheet applications (Microsoft Excel, LibreOffice Calc) are very powerful tools for dealing with data. They feature pivot tables for exploration, solvers for optimization problems and connectors to pull data from any database. Needless to say that most data analytics today is done in a spreadsheet application.

Our client, a renowned research institute, collects data from many third-party businesses and analyze their environmental impact. Up to now, the institute sends a spreadsheet questionnaire to each business by mail. Each has to fill in the values and send it back. Afterwards, the analysis is done in a series of spreadsheets as well.

When scaling up this approach, the researchers faced problems with data collection (a lot of back and forth emailing because of missing/invalid values) and in-house collaboration (having slightly different formulas in each project makes comparing results difficult, sharing/reusing data is cumbersome because of the heterogenous schema of the collection spreadsheets). Also consolidation of analysis logic from many spreadsheets into a single one is an elaborate, error-prone task.

## Separation of data and logic is the key to success
Facing the above problems, our client came to the conclusion to implement a more elaborate system. And this is where we came into play.

The main challenges were:

1. Enable easy sharing of data between projects while still being flexible like a spreadsheet application (easily introduce new variables).
1. Allow reuse of analysis logic by separating it from the data while still satisfying result reproducibility.
1. Simplify data collection and validation.

All three challenges are worth a separate blog entry. Today, I focus on the second challenge but I will also say a few words about the others.

The first challenge we addressed by using a so-called SQL antipattern, the [entity-attribute-value model](https://en.wikipedia.org/wiki/Entity%E2%80%93attribute%E2%80%93value_model).
To enable data reuse between projects, a global catalog of variables is introduced. The variable definition includes the data type (string, number, date, ..), the physical unit and minimum / maximum values (numbers only). If necessary, new variables can be defined quickly.

The analysis (the second challenge) is no longer placed into spreadsheets but is written in Python code stored in Git repositories. These so called *analysis modules* enable code reuse between projects. To ensure reproducibility (=we want to use the exact same code everytime), analysis modules are referenced by their Git commit. A project can utilize multiple analysis modules. This allows them to build a versatile toolbox of modules (i.e. data preparation, calculation, allocation to key performance values).
In order to get meaningful results we need to know in which order the modules need to be executed. For this we create a schedule from a dependency graph based on the input and output variables of each module (See next section). The dependency graph is also crucial for creating the questionnaire in the third challenge.

For the third challenge, we built a web app that automatically creates online questionnaires based on the variables required by the project.

The app consists of a frontend, written in Angular, and a REST-style API backend, written in C# (ASP.NET Core 2.1). In addition we operate an ElasticSearch instance to provide fuzzy search for the variables and we implemented an endpoint to fetch data for ad-hoc analysis and module development in [Jupyter Notebooks](https://jupyter.org/). What you cannot see in the following diagram is that the backend is further divided into *API* and *analytics engine*. There is no communication between them except via the database. This not only reduces the spaghetti force but it also allows us to scale the analytics engine horizontally if needed.

![A diagram of the high level components frontend, python subsystem, core backend, elasticsearch and the database](/assets/2019-06-20/ComponentOverview.svg)

In the remaining blog post, I'll focus on dependency analysis and execution of modules. [Let me know](https://twitter.com/eliogubser) if you are interested in another blog post about other parts of the system.

## Dependency analysis and execution schedule
Consider a project that utilizes the four modules A, B, C and D as seen in the diagram (way) below. Our goal is to know two things:

<style type="text/css">
    ol.alphabetic { list-style-type: lower-alpha; }
</style>

<ol class="alphabetic">
  <li>The set of variables that we need to add to the questionnaire. (third challenge)</li>
  <li>A feasible execution schedule that satisfies the input and output dependencies of the modules. (second challenge)</li>
</ol>

The whole process is summarized in this diagram and explained in the following sections:
![From Python code to a schedule](/assets/2019-06-20/FromModuleToSchedule.svg)

### Determine dependencies of a single module
Let's start by detecting the input and output variables in the Python code. We require that an analysis module has to follow the rule that inputs are attributes on an object called `inp` and outputs are attributes on an object called `out`. Example:
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
  <li>By walking the whole tree starting from the end node, we can determine the order of execution (= a feasible schedule).</li>
</ol>

---

👉 This post is also featured on [geo.ebp.ch](https://geo.ebp.ch).

