---
title: "Introduction iRODS Python API"
teaching: 30
exercises: 0
questions:
- "How to do basic file handling with the iRODS Python API"
- "How to use the iRODS Python API in data provenance"
objectives:
- "Connect to iRODS via Python API"
- "Upload/Download data objects"
- "Create and run through collections"
- "Metadata handling of data objects and collections"
- "Querying data objects and collections based on metadata"
keypoints:
- "All iRODS objects or Python objects"
- "Querying is tricky"
---

This episodes describes how to interact with your data in iRODS via the Python API.

## Connect to iRODS

We can connect to our iRODS instance by passing our credentials to a session object.
For this we need to import the iRODS session library.
However, we also want to savely pass our password.
Therefore, we will also import the getpass library.
It would even be better if we could have TOTP connected to our iRODS account. 
However, this is not possible with iRODS yet.
After importing the correct libraries, we can safely store our password and send our credentials to iRODS.

~~~
import getpass
from irods.session import iRODSSession

pw = getpass.getpass().encode('base64')
session = iRODSSession(host='<hostname>', port=1247, user='<username>', password=pw.decode('base64'),zone='<zoneName>)'
~~~
{: .language-python}

We can test whether our connection succeeded by trying to get our home folder:
~~~
coll = session.collections.get('/aliceZone/home/irods-user1')
print(coll.data_objects)
print(coll.subcollections)
~~~
{: .language-python}

So far no data is stored in your iRODS collection.
Let us upload some data.

We will need our homefolder more often as a reference point, so let us store the collection path:

~~~
iHome = coll.path
~~~
{: .language-python}


## GitHub Pages

If a repository has a branch called `gh-pages` (short for "GitHub Pages"),
GitHub publishes its content to create a website for the repository.
If the repository's URL is `https://github.com/USERNAME/REPOSITORY`,
the website is `https://USERNAME.github.io/REPOSITORY`.

GitHub Pages sites can include static HTML pages,
which are published as-is,
or they can use [Jekyll][jekyll] as described below
to compile HTML and/or Markdown pages with embedded directives
to create the pages for display.

> ## Why Doesn't My Site Appear?
>
> If the root directory of a repository contains a file called `.nojekyll`,
> GitHub will *not* generate a website for that repository's `gh-pages` branch.
{: .callout}

We write lessons in Markdown because it's simple to learn
and isn't tied to any specific language.
(The ReStructured Text format popular in the Python world,
for example,
is a complete unknown to R programmers.)
If authors want to write lessons in something else,
such as [R Markdown][r-markdown],
they must generate HTML or Markdown that [Jekyll][jekyll] can process
and commit that to the repository.
A [later episode]({{ page.root }}/04-formatting/) describes the Markdown we use.

> ## Teaching Tools
>
> We do *not* prescribe what tools instructors should use when actually teaching:
> the [Jupyter Notebook][jupyter],
> [RStudio][rstudio],
> and the good ol' command line are equally welcome up on stage.
> All we specify is the format of the lesson notes.
{: .callout}

## Jekyll

GitHub uses [Jekyll][jekyll] to turn Markdown into HTML.
It looks for text files that begin with a header formatted like this:

~~~
---
variable: value
other_variable: other_value
---
...stuff in the page...
~~~
{: .source}

and inserts the values of those variables into the page when formatting it.
The three dashes that start the header *must* be the first three characters in the file:
even a single space before them will make [Jekyll][jekyll] ignore the file.

The header's content must be formatted as [YAML][yaml],
and may contain Booleans, numbers, character strings, lists, and dictionaries of name/value pairs.
Values from the header are referred to in the page as `page.variable`.
For example,
this page:

~~~
---
name: Science
---
{% raw %}Today we are going to study {{page.name}}.{% endraw %}
~~~
{: .source}

is translated into:

~~~
<html>
  <body>
    <p>Today we are going to study Science.</p>
  </body>
</html>
~~~
{: .html}

> ## Back in the Day...
>
> The previous version of our template did not rely on Jekyll,
> but instead required authors to build HTML on their desktops
> and commit that to the lesson repository's `gh-pages` branch.
> This allowed us to use whatever mix of tools we wanted for creating HTML (e.g., [Pandoc][pandoc]),
> but complicated the common case for the sake of uncommon cases,
> and didn't model the workflow we want learners to use.
{: .callout}

## Configuration

[Jekyll][jekyll] also reads values from a configuration file called `_config.yml`,
which are referred to in pages as `site.variable`.
The [lesson template]({{ site.template_repo }}) does *not* include `_config.yml`,
since each lesson will change some of its value,
which would result in merge collisions each time the lesson was updated from the template.
Instead,
the [template]({{ site.template_repo }}) contains a script called `bin/lesson_initialize.py`
which should be run *once* to create an initial `_config.yml` file
(and a few other files as well).
The author should then edit the values in the top half of the file.

## Collections

If several Markdown files are stored in a directory whose name begins with an underscore,
[Jekyll][jekyll] creates a [collection][jekyll-collection] for them.
We rely on this for both lesson episodes (stored in `_episodes`)
and extra files (stored in `_extras`).
For example,
putting the extra files in `_extras` allows us to populate the "Extras" menu pulldown automatically.
To clarify what will appear where,
we store files that appear directly in the navigation bar
in the root directory of the lesson.
[The next episode]({{ page.root }}/03-organization/) describes these files.

{% include links.md %}
