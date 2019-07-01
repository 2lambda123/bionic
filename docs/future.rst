====================
The Future of Bionic
====================

Development Status
-------------------

Bionic is still at an early stage, and many features have been planned but not
implemented.  All of these features should be developed at some point, but the
exact timeline is not fixed.  You can help us prioritize by `giving us feedback
<get-help.rst>`_!

Future Work
-----------

Local Parallelization
.....................

Currently Bionic computes everything in a single process.  Later it will be
able to run multiple tasks on different processes in parallel, allowing you to
take advantage of all the CPU cores of whatever machine you're using.

Distributed Computation
.......................

Similar to the above: Bionic will eventually be able to dispatch jobs to other
machines (such as a cloud-based compute cluster) to achieve even more
parallelization.

Cloud Storage
.............

Currently Bionic can only persist data to the local hard drive of the machine
it's running on.  Later it will be able to save to shared cloud storage, such
as `Google Cloud Storage <https://cloud.google.com/storage/>`_.

Flow Merging and Nesting
........................

Bionic will provide better APIs to combine multiple flows into one, or to nest
one flow inside another with its own namespace.  This should enable the
construction of more complex and/or more reusable flows.

Direct Access to Persisted Files
................................

Bionic is built around the idea that the user's code generally wants to operate
on in-memory objects rather than files.  However, in some cases it's preferable
to operate on the raw files.  For example, if a file is large we might want to
load only small parts into memory at a time; or we might want to call an
external script that only knows how to operate on files.  In these cases it
would be helpful to be able to do something like this:

.. code-block:: python

    @builder
    @bionic.arg_as_file_path('raw_frame', 'raw_frame_path')
    def transformed_data(raw_frame_path):
        assert raw_frame_path.suffix == '.pq'
        subprocess.check_call(['transform_data.sh', str(raw_frame_path)])

Graph-Rewriting Decorators
..........................

Normally Bionic translates each entity into a single node (or a parallel set of
nodes) in its dependency graph.  However, in somes cases we might want to
generate a more complex subgraph.  For example, the author of an entity might
know that its computation can be safely broken into chunks and run in parallel:

.. code-block:: python

    @builder
    @bionic.parallelize_by_row('raw_frame'):
    def filtered_data(raw_frame, relevant_categories):
        return raw_frame[raw_frame['category'].isin(relevant_categories)]


User-Defined Decorators
.......................

Bionic currently provides several built-in decorators, but their implementation
is complex and tightly coupled with Bionic's internals.  This is partly because
we're still figuring out what Bionic's internal data model should look like.
Once those internals are cleaner and more stable, it will be possible for users
to write (and share) their own decorators.

For example, Bionic provides a built-in :func:`@pyplot <bionic.pyplot>`
decorator to make Matplotlib plotting easier.  We might want similar decorators
for other external libraries that are awkward to use in the Bionic framework.

Smarter Cache Invalidation
..........................

Although Bionic attempts to automatically figure out when cached data can be
used and when it needs to be recomputed, the user still needs to tell it about
code changes using :func:`@version <bionic.version>`.  This requirement
probably can't be completely eliminated, since there are many factors that
might affect our computation (other functions, other libraries, our operating
system, external databases), and even if we could detect when any of them
changed, the changes might or might not be subjectively meaningful.  For
example, if the user performs a minor refactoring, they might not want to
recompute their results.  However, in common cases we can at least warn the
user when they might have forgotten to re-version their code.

In particular, we can usually (but not always) detect when an entity function
changes, and force the user to indicate whether the change is meaningful or
not.  This might involve a concept of a "major" and a "minor" version, where
the major version update indicates a functional change that invalidates old
results, while a minor version update indicates a non-functional change.

Automatic Regression Tests
..........................

Following up on the concept of non-functional changes above: when a user
performs a change that is supposed to be non-functional, they might actually
want Bionic to verify this by re-running their code and confirming that the
output is the same as the previous version's.

Data Validation
...............

Often we'd like to make assertions about an entity's output and be alerted if
those assertions are violated.  Currently this can be done in two ways: adding
``assert`` statements to the entity's function, or writing
a custom `Protocol <api/protocols.rst>`_ with a special ``validate`` method.
These solutions share two problems.  First, they have to be written by the
person who defines the entity; it's not possible to add new assertions about
pre-exsiting entities.  Second, if the assertions fail, the entity's value
never gets persisted, so it's difficult to debug the problem -- especially if
the value was expensive to compute.

A better approach would be a first-class concept of an entity that validates
other entities, after their value has been persisted but before it can be
consumed by any other (non-validator) entities.

Better Multiplicity Abstractions
................................

Bionic's concept of creating multiple values for an entity and then gathering
them together is fairly novel (as far as we know), which means it will probably
require some iteration before we find the best way to work with it.  There are
definitely many use cases of multiplicity that are awkward or impossible to
express with the current API.  For example, we might want one entity to be able
to generate multiple downstream instances of another: for example, a
``hyperparameter_search_strategy`` entity which creates multiple instances of a
``hyperparameters_dict`` entity.