Configuration
=============

The job definitions for Jenkins Job Builder are kept in any number of
YAML files, in whatever way you would like to organize them.  When you
invoke ``jenkins-jobs`` you may specify either the path of a single
YAML file, or a directory.  If you choose a directory, all of
the .yaml (or .yml) files in that directory will be read, and all the
jobs they define will be created or updated.

Definitions
-----------

Jenkins Job Builder understands a few basic object types which are
described in the next sections.

.. _job:

Job
^^^

The most straightforward way to create a job is simply to define a
Job in YAML.  It looks like this::

  - job:
      name: job-name

That's not very useful, so you'll want to add some actions such as
:ref:`builders`, and perhaps :ref:`publishers`.  Those are described
later.  There are a few basic optional fields for a Job definition::

  - job:
      name: job-name
      project-type: freestyle
      defaults: global
      disabled: false
      display-name: 'Fancy job name'
      concurrent: true
      quiet-period: 5
      workspace: /srv/build-area/job-name
      block-downstream: false
      block-upstream: false

**project-type**
  Defaults to "freestyle", but "maven" as well as "multijob" or "flow"
  can also be specified.

**defaults**
  Specifies a set of `Defaults`_ to use for this job, defaults to
  ''global''.  If you have values that are common to all of your jobs,
  create a ``global`` `Defaults`_ object to hold them, and no further
  configuration of individual jobs is necessary.  If some jobs
  should not use the ``global`` defaults, use this field to specify a
  different set of defaults.

**disabled**
  Boolean value to set whether or not this job should be disabled in
  Jenkins. Defaults to ``false`` (job will be enabled).

**display-name**
  Optional name shown for the project throughout the Jenkins web GUI in place
  of the actual job name.  The jenkins_jobs tool cannot fully remove this trait
  once it is set, so use caution when setting it.  Setting it to the same
  string as the project name is an effective un-set workaround.  Alternately,
  the field can be cleared manually using the Jenkins web interface.

**concurrent**
  Boolean value to set whether or not Jenkins can run this job
  concurrently. Defaults to ``false``.

**quiet-period**
  Number of seconds to wait between consecutive runs of this job.
  Defaults to ``0``.

**workspace**
  Path for a custom workspace. Defaults to Jenkins default
  configuration.

**block-downstream**
  Boolean value to set whether or not this job must block while
  downstream jobs are running. Downstream jobs are determined
  transitively. Defaults to ``false``.

**block-upstream**
  Boolean value to set whether or not this job must block while
  upstream jobs are running. Upstream jobs are determined
  transitively. Defaults to ``false``.

**auth-token**
  Specifies an authentication token that allows new builds to be
  triggered by accessing a special predefined URL. Only those who
  know the token will be able to trigger builds remotely.

.. _job-template:

Job Template
^^^^^^^^^^^^

If you need several jobs defined that are nearly identical, except
perhaps in their names, SCP targets, etc., then you may use a Job
Template to specify the particulars of the job, and then use a
`Project`_ to realize the job with appropriate variable substitution.

A Job Template has the same syntax as a `Job`_, but you may add
variables anywhere in the definition.  Variables are indicated by
enclosing them in braces, e.g., ``{name}`` will substitute the
variable `name`.  When using a variable in a string field, it is good
practice to wrap the entire string in quotes, even if the rules of
YAML syntax don't require it because the value of the variable may
require quotes after substitution. In the rare situation that you must
encode braces within literals inside a template (for example a shell
function definition in a builder), doubling the braces will prevent
them from being interpreted as a template variable.

You must include a variable in the ``name`` field of a Job Template
(otherwise, every instance would have the same name).  For example::

  - job-template:
      name: '{name}-unit-tests'

Will not cause any job to be created in Jenkins, however, it will
define a template that you can use to create jobs with a `Project`_
definition.  It's name will depend on what is supplied to the
`Project`_.

.. _project:

Project
^^^^^^^

The purpose of a project is to collect related jobs together, and
provide values for the variables in a `Job Template`_.  It looks like
this::

  - project:
      name: project-name
      jobs:
        - '{name}-unit-tests'

Any number of arbitrarily named additional fields may be specified,
and they will be available for variable substitution in the job
template.  Any job templates listed under ``jobs:`` will be realized
with those values.  The example above would create the job called
'project-name-unit-tests' in Jenkins.

The ``jobs:`` list can also allow for specifying job-specific
substitutions as follows::

  - job-template:
      name: '{name}-unit-tests'
      builders:
        - shell: unittest
      publishers:
        - email:
            recipients: '{mail-to}'

  - job-template:
      name: '{name}-perf-tests'
      builders:
        - shell: perftest
      publishers:
        - email:
            recipients: '{mail-to}'

  - project:
      name: project-name
      jobs:
        - '{name}-unit-tests':
            mail-to: developer@nowhere.net
        - '{name}-perf-tests':
            mail-to: projmanager@nowhere.net

If a variable is a list, the job template will be realized with the
variable set to each value in the list.  Multiple lists will lead to
the template being realized with the cartesian product of those
values.  Example::

  - project:
      name: project-name
      pyver:
       - 26
       - 27
      jobs:
       - '{name}-{pyver}'

Job Group
^^^^^^^^^

If you have several Job Templates that should all be realized
together, you can define a Job Group to collect them.  Simply use the
Job Group where you would normally use a `Job Template`_ and all of
the Job Templates in the Job Group will be realized.  For example::

  - job-template:
      name: '{name}-python-26'

  - job-template:
      name: '{name}-python-27'

  - job-group:
      name: python-jobs
      jobs:
        - '{name}-python-26'
        - '{name}-python-27'

  - project:
      name: foo
      jobs:
        - python-jobs

Would cause the jobs `foo-python-26` and `foo-python-27` to be created
in Jenkins.

The ``jobs:`` list can also allow for specifying job-specific
substitutions as follows::

  - job-group:
      name: job-group-name
      jobs:
        - '{name}-build':
            pipeline-next: '{name}-upload'
        - '{name}-upload':
            pipeline-next: ''

.. _macro:

Macro
^^^^^

Many of the actions of a `Job`_, such as builders or publishers, can
be defined as a Macro, and then that Macro used in the `Job`_
description.  Builders are described later, but let's introduce a
simple one now to illustrate the Macro functionality.  This snippet
will instruct Jenkins to execute "make test" as part of the job::

  - job:
      name: foo-test
      builders:
        - shell: 'make test'

If you wanted to define a macro (which won't save much typing in this
case, but could still be useful to centralize the definition of a
commonly repeated task), the configuration would look like::

  - builder:
      name: make-test
      builders:
        - shell: 'make test'

  - job:
      name: foo-test
      builders:
        - make-test

This allows you to create complex actions (and even sequences of
actions) in YAML that look like first-class Jenkins Job Builder
actions.  Not every attribute supports Macros, check the documentation
for the action before you try to use a Macro for it.

Macros can take parameters, letting you define a generic macro and more
specific ones without having to duplicate code::

    # The 'add' macro takes a 'number' parameter and will creates a
    # job which prints 'Adding ' followed by the 'number' parameter:
    - builder:
        name: add
        builders:
         - shell: "echo Adding {number}"

    # A specialized macro 'addtwo' reusing the 'add' macro but with
    # a 'number' parameter hardcoded to 'two':
    - builder:
        name: addtwo
        builders:
         - add:
            number: "two"

    # Glue to have Jenkins Job Builder to expand this YAML example:
    - job:
        name: "testingjob"
        builders:
         # The specialized macro:
         - addtwo
         # Generic macro call with a parameter
         - add:
            number: "ZERO"
         # Generic macro called without a parameter. Never do this!
         # See below for the resulting wrong output :(
         - add

Then ``<builders />`` section of the generated job show up as::

  <builders>
    <hudson.tasks.Shell>
      <command>echo Adding two</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo Adding ZERO</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo Adding {number}</command>
    </hudson.tasks.Shell>
  </builders>

As you can see, the specialized macro ``addtwo`` reused the definition from
the generic macro ``add``.  Whenever you forget a parameter from a macro,
it will not be expanded and left as is, which will most probably cause havoc in
your Jenkins builds.

Lists and Dictionaries can also be passed as arguments to a macro::

    - builder:
        name: pass-list-example
        # The variable must span the whole string for this to work
        builders: '{var}'
        
    - builder:
        name: pass-dict-example
        builders:
            - maven-target: '{var}'
        
    - job:
        name: macro-example
        builders:
            - pass-list-example:
                # var contains a list
                var:
                    - shell: echo hi
                    - shell: echo hello
            - pass-dict-example:
                # var contains a dictionary
                var:
                    goals: deploy
            
Result::

    <builders>
        <hudson.tasks.Shell>
          <command>echo hi</command></hudson.tasks.Shell>
        <hudson.tasks.Shell>
          <command>echo hello</command>
        </hudson.tasks.Shell>
        <hudson.tasks.Maven>
          <targets>deploy</targets>
          ...
    </builders>

.. _defaults:

Defaults
^^^^^^^^

Defaults collect job attributes (including actions) and will supply
those values when the job is created, unless superseded by a value in
the 'Job'_ definition.  If a set of Defaults is specified with the
name ``global``, that will be used by all `Job`_ (and `Job Template`_)
definitions unless they specify a different Default object with the
``defaults`` attribute.  For example::

  - defaults:
      name: global
      description: 'Do not edit this job through the web!'

Will set the job description for every job created.

Python Expressions
^^^^^^^^^^^^^^^^^^

A python expression can be embedded in a string by surrounding it with the characters '<?' and '>'::

    - job:
        name: python-expression
        builders:
            - shell: <?'echo ' + str(5 + 3)>
            
Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo 8</command></hudson.tasks.Shell>
  </builders>
      
Variables within the expression get replaced before the expression is evaluated::

    - job-template:
        name: python-expression-{var}
        builders:
            - shell: <?'echo ' + '{var}'>

    - project:
        name: python-expression-var
        var:
            - hi
        jobs:
            - python-expression-{var}

Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo hi</command></hudson.tasks.Shell>
  </builders>
      
Because variables can be used, the characters '{' and '}' must be escaped in order to be used::

    - job:
        name: python-expression-braces
        builders:
            # Characters '{' and '}' are doubled to be escaped
            - shell: "<?'echo ' + str({{'key' : 'val'}})>"
            
Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo {'key': 'val'}</command></hudson.tasks.Shell>
  </builders>
      
If the '>' character is needed within the expression, it can be escaped using a backslash. Of course it is encoded as &gt; in the resulted xml::

    - job:
        name: python-expression-gt
        builders:
            - shell: <?'cat file1.txt \> file2.txt '>

Result::

  <builders>
    <hudson.tasks.Shell>
      <command>cat file1.txt &gt; file2.txt </command></hudson.tasks.Shell>
  </builders>
      
Expressions can also return a list or a dictionary::

    - job:
        name: python-expression-list
        # Returning a list
        # The expression must span the whole string for this to work
        builders: "<?[{{'shell': 'echo hi'}}]>"

Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo hi</command></hudson.tasks.Shell>
  </builders>
      
A list can be flattened in its parent list if using an extra question mark at the beginning of the expression::

    - job:
        name: python-expression-flattened
        builders:
            # Notice the double question marks
            - "<??[{{'shell': 'echo hi'}}]>"
            - shell: echo hello

Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo hi</command></hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo hello</command>
    </hudson.tasks.Shell>
  </builders>
      
External Python Functions
^^^^^^^^^^^^^^^^^^^^^^^^^

External python functions can be added in .py files.
Those files must be in the same directory containing the .yaml (.yml) files.

example.py::

    # 'test' will be accessible through the 'example' variable since it comes from a file named example.py
    def test(str):
        return 'echo ' + str
        
example.yaml::

    - job:
        name: python-expression-external
        builders:
            - shell: <?example.test('hi')>
      
Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo hi</command></hudson.tasks.Shell>
  </builders>

Contexts
^^^^^^^^

A context can be attached to a job template. It contains configurations that get folded based on a variable. This can probably be better explained with an example::

    - context:
        name: simple-context
        # The name of the variable that will be used for the folding
        var-name: branch
        configs:
            branch-name:
                # The keys at this level are compared with the value contained by the variable in var-name and the value of the matching one is kept and put directly in branch-name
                1.0: trunk
                2.0: branches/context-example-2.0

    - job-template:
        name: context-example-{branch}
        context: simple-context
        builders:
            - shell: <?'echo ' + configs['branch-name']>

    - project:
        name: context-example
        branch:
            - 1.0
            - 2.0
        jobs:
            - context-example-{branch}

The context gets folded when each instance of the job template gets generated, so a different result is generated each time.
The resulting context looks like this for each of the job templates generated.

Folded result for context-example-1.0::

    configs:
        branch-name: trunk
        
Folded result for context-example-2.0::

    configs:
        branch-name: branches/context-example-2.0
        
That context can then be accessed in a python expression as in the example with configs['config-name']

Result for context-example-1.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo trunk</command></hudson.tasks.Shell>
  </builders>
  
Result for context-example-2.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo branches/context-example-2.0</command></hudson.tasks.Shell>
  </builders>
  
A context can also have children contexts attached to it. A tree gets built based on this::

    - context:
        name: child
        var-name: branch
        configs:
            branch-name:
                1.0: branches/context-example-child-1.0

    - context:
        name: parent
        var-name: branch
        children:
            child:
        configs:
            branch-name:
                1.0: branches/context-example-parent-1.0

    - job-template:
        name: context-example-child-{branch}
        context: child
        builders:
            - shell: "<?'echo my branch: ' + configs['branch-name']>"
            - shell: "<?'echo my parent''s branch: ' + parents['parent'][0]['branch-name']>"

    - job-template:
        name: context-example-parent-{branch}
        context: parent
        builders:
            - shell: "<?'echo my branch: ' + configs['branch-name']>"
            - shell: "<?'echo my child''s branch: ' + children['child']['branch-name']>"

    - project:
        name: context-example-children
        branch:
            - 1.0
        jobs:
            - context-example-child-{branch}
            - context-example-parent-{branch}

Folded result for context-example-child-1.0::

    configs:
        branch-name: branches/context-example-child-1.0
    parents:
        parent:
            -
                branch-name: branches/context-example-parent-1.0
                parents:
    children:
    
Folded result for context-example-parent-1.0::

    configs:
        branch-name: branches/context-example-parent-1.0
    parents:
    children:
        child:
            branch-name: branches/context-example-child-1.0
            children:
            
You might notice how the 'parents' entry in context-example-child-1.0 contains a list of entries for each parent. This is because a child can have many times the same parent, as we'll see in the next example.

Result for context-example-child-1.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my branch: branches/context-example-child-1.0</command></hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo my parents branch: branches/context-example-parent-1.0</command>
    </hudson.tasks.Shell>
  </builders>
  
Result for context-example-parent-1.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my branch: branches/context-example-parent-1.0</command></hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo my childs branch: branches/context-example-child-1.0</command>
    </hudson.tasks.Shell>
  </builders>

A mapping of values between the parent and child contexts can also be defined::

    - context:
        name: child-mapping
        var-name: branch
        configs:
            branch-name:
                1.0: branches/context-example-child-1.0

    - context:
        name: parent-mapping
        var-name: branch
        children:
            child-mapping:
                # The name of the entry here must correspond to var-name.
                branch:
                    1.0: 1.0
                    2.0: 1.0
        configs:
            branch-name:
                1.0: branches/context-example-parent-1.0
                2.0: branches/context-example-parent-2.0

    - job-template:
        name: context-example-mapping-child-{branch}
        context: child-mapping
        builders:
            - shell: "<?'echo my branch: ' + configs['branch-name']>"
            - shell: "<?'echo my parent''s branches: ' + str([parent['branch-name'] for parent in parents['parent-mapping']])>"

    - project:
        name: context-example-mapping
        branch:
            - 1.0
        jobs:
            - context-example-mapping-child-{branch}

Folded result::

    configs:
        branch-name: branches/context-example-child-1.0
    parents:
        parent:
            -
                branch-name: branches/context-example-parent-1.0
                parents:
            -
                branch-name: branches/context-example-parent-2.0
                parents:
    children:

Because both 'branch' values for the parent map to the value 1.0 for the child, the child ends up having twice the same parent, once for each value.
    
Result::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my branch: branches/context-example-child-1.0</command></hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo my parents branches: ['branches/context-example-parent-1.0', 'branches/context-example-parent-2.0']</command>
    </hudson.tasks.Shell>
  </builders>

A default value can also be specified in a context config::

    - context:
        name: context-default
        var-name: branch
        configs:
            branch-name:
                1.0: trunk
                # The 'default' entry gets used when no other entry matches the value of the context variable specified by var-name
                # Also notice the usage of the {branch} variable which gets replaced by its corresponding value
                default: branches/context-example-default-{branch}

    - job-template:
        name: context-example-default-{branch}
        context: context-default
        builders:
            - shell: "<?'echo my branch: ' + configs['branch-name']>"

    - project:
        name: context-example-default
        branch:
            - 1.0
            - 2.0
        jobs:
            - context-example-default-{branch}

Folded result for context-example-default-1.0::

    configs:
        branch-name: trunk
        
Folded result for context-example-default-2.0::

    configs:
        branch-name: branches/context-example-default-2.0
        
Result for context-example-default-1.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my branch: trunk</command></hudson.tasks.Shell>
  </builders>

Result for context-example-default-2.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my branch: branches/context-example-default-2.0</command></hudson.tasks.Shell>
  </builders>

Defaults shouldn't be used in the mapping of child values. For example, don't do this::

    - context:
        name: parent-mapping
        var-name: branch
        children:
            child-mapping:
                branch:
                    1.0: 1.0
                    # This is bad
                    default: 2.0

Determining the parent values for 'branch' when a child context is evaluated is not possible in that case since default can be any value.

Extra child configs can be added in the declaration of a child::

    - context:
        name: extra-child
        var-name: branch
        configs:
            branch-name:
                1.0: branches/context-example-extra-child-1.0
                2.0: branches/context-example-extra-child-2.0

    - context:
        name: extra-parent
        var-name: branch
        children:
            extra-child:
                branch:
                    1.0: 1.0
                    2.0: 2.0
                # Here is an extra config
                build-child:
                    1.0: false
                    2.0: true
        configs:
            branch-name:
                1.0: branches/context-example-extra-parent-1.0
                2.0: branches/context-example-extra-parent-2.0

    - job-template:
        name: context-example-extra-parent-{branch}
        context: extra-parent
        builders:
            - shell: "<?'echo my child''s branch: ' + children['extra-child']['branch-name'] + ' and I should ' + ('' if children['extra-child']['build-child'] else 'not ') + 'build it'>"

    - project:
        name: context-example-extra
        branch:
            - 1.0
            - 2.0
        jobs:
            - context-example-extra-parent-{branch}

Folded result for context-example-extra-parent-1.0::

    configs:
        branch-name: branches/context-example-extra-parent-1.0
    parents:
    children:
        child:
            branch-name: branches/context-example-extra-child-1.0
            build-child: false
            children:
            
Folded result for context-example-extra-parent-2.0::

    configs:
        branch-name: branches/context-example-extra-parent-2.0
    parents:
    children:
        child:
            branch-name: branches/context-example-extra-child-2.0
            build-child: true
            children:
            
Result for context-example-extra-parent-1.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my childs branch: branches/context-example-extra-child-1.0 and I should not build it</command></hudson.tasks.Shell>
  </builders>
  
Result for context-example-extra-parent-2.0::

  <builders>
    <hudson.tasks.Shell>
      <command>echo my childs branch: branches/context-example-extra-child-2.0 and I should build it</command></hudson.tasks.Shell>
  </builders>

Modules
-------

The bulk of the job definitions come from the following modules.

.. toctree::
   :maxdepth: 2

   project_flow
   project_freestyle
   project_maven
   project_matrix
   general
   builders
   hipchat
   metadata
   notifications
   parameters
   properties
   publishers
   reporters
   scm
   triggers
   wrappers
   zuul


Module Execution
----------------

The jenkins job builder modules are executed in sequence.

Generally the sequence is:
    #. parameters/properties
    #. scm
    #. triggers
    #. wrappers
    #. prebuilders (maven only, configured like :ref:`builders`)
    #. builders (maven, freestyle, matrix, etc..)
    #. postbuilders (maven only, configured like :ref:`builders`)
    #. publishers/reporters/notifications

