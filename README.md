# Code Maat

Code Maat is a command line tool used to mine and analyze data from version-control systems (VCS).

## The ideas behind Code Maat

To understand large-scale software systems we need to look at their evolution. The history of our system provides us with data we cannot derive from a single snapshot of the source code. Instead VCS data blends technical, social and organizational information along a temporal axis that let us map out our interaction patterns in the code. Analyzing these patterns gives us early warnings on potential design issues and development bottlenecks, as well as suggesting new modularities based on actual interactions with the code. Addressing these issues saves costs, simplifies maintenance and let us evolve our systems in the direction of how we actually work with the code.

Code Maat was developed to accompany the discussions in my book [Code as a Crime Scene](https://leanpub.com/crimescene).

### About the name

Maat was a goddess in ancient Egyptian myth. She was the one who gave us order out of the initial chaos in the universe. Code Maat hopes to continue the work of Maat, albeit on a smaller basis, by highlighting code with chaotic development practices and suggest the directions of future refactorings to bring order to it. Further, maat was used in ancient Egypt as a concept of truth. And metrics never lie (except when they do).

## License

Copyright © 2013 Adam Tornhill

Distributed under the [GNU General Public License v3.0](http://www.gnu.org/licenses/gpl.html).

## Usage

Currently, I'm not hosting any pre-built binaries. Code Maat is written in Clojure. To build it from source, use [leiningen](https://github.com/technomancy/leiningen):

	   lein uberjar

The command above will create a standalone `jar` containing all the dependencies.

Code Maat operates on log files from version-control systems. The supported version-control systems are `git`, Mercurial (`hg`) and `svn`. The log files are generated by using the version-control systems themselves as described in the following sections.

### Generating input data

#### Preparations

To analyze our VCS data we need to define a temporal period of interest. Over time, many design issues do get fixed and we don't want old data to infer with our current analysis of the code. To limit the data Code Maat will consider, use one of the following flags depending on your version-control system:
+ *git:* Use the `--after=<date>` to specify the last date of interest. The `<date>` is given as `YYYY-MM-DD`.
+ *hg:* Ue the `--date` swith to specify the last date of interest. The value is given as `">YYYY-MM-DD"`.
+ *svn:* Use the `-r` option to specify a range of interest, for example `-r {20130820}:HEAD`.

#### Generate a Subversion log file using the following command:

          svn log -v --xml > logfile.log -r {YYYYmmDD}:HEAD

#### Generate a git log file using the following command:

          git log --pretty=format:'[%h] %an %ad %s' --date=short --numstat --after=YYYY-MM-DD

#### Generate a Mercurial log file using the following command:

          hg log --template "rev: {rev} author: {author} date: {date|shortdate} files:\n{files %'{file}\n'}\n" --date ">YYYY-MM-DD"

### Running Code Maat

You can run Code Maat directly from leiningen:

    	  lein run logfile.log -vcs <vcs>

If you've built a standalone jar (`lein uberjar`), run it with a simple java invocation:

     	  java -jar code-maat-0.2.0.jar -vcs <vcs>

When invoked without any arguments, Code Maat prints its usage:

             adam$ java -jar code-maat-0.2.0.jar
             Switches                 Default   Desc
             --------                 -------   ----
             -vcs, --version-control            Input vcs module type: supports svn, git or hg
             -a,  --analysis           authors  The analysis to run (authors, revisions, coupling, summary, identity)
             -r, --rows                10       Max rows in output
             --min-revs                5        Minimum number of revisions to include an entity in the analysis
             --min-shared-revs         5        Minimum number of shared revisions to include an entity in the analysis
             --min-coupling            50       Minimum degree of coupling (in percentage) to consider
             --max-coupling           100       Maximum degree of coupling (in percentage) to consider

#### Generating a summary

When starting out, I find it useful to get an overview of the mined data. With the `summary` analysis, Code Maat produces such an overview:

   	   java -jar code-maat-0.2.0.jar logfile.log -vcs git -a summary

The resulting output is on csv format:

              statistic,                 value
              number-of-commits,           919
              number-of-entities,          730
              number-of-entities-changed, 3397
              number-of-authors,            79

#### Mining organizational metrics

By default, Code Maat runs an analysis on the number of authors per module. The authors analysis is based on the idea that the more developers working on a module, the larger the communication challenges. The analysis is invoked with the following command:

   	   java -jar code-maat-0.2.0.jar logfile.log -vcs git

The resulting output is on CSV format:

              entity,         n-authors, n-revs
              InfoUtils.java, 12,        60
              BarChart.java,   7,        30
              Page.java,       4,        27
              ...

In example above, the first column gives us the name of module, the second the total number of distinct authors that have made commits on that module, and the third column gives us the total number of revisions of the module. Taken together, these metrics serve as predictors of defects and quality issues.

#### Mining logical coupling

Logical coupling refers to modules that tend to change together. Modules that are logically coupled have a hidden, implicit dependency between them such that a change to one of them leads to a predictable change in the coupled module. To analyze the logical coupling in a system, invoke Code Maat with the following arguments:

              java -jar code-maat-0.1.0.jar logfile.log -vcs git -a coupling

The resulting output is on CSV format:

              entity,          coupled,        degree,  average-revs
              InfoUtils.java,  Page.java,      78,      44
              InfoUtils.java,  BarChart.java,  62,      45
              ...

In the example above, the first column (`entity`) gives us the name of the module, the second (`coupled`) gives us the name of a logically coupled module, the third column (`degree`) gives us the coupling as a percentage (0-100), and finally `average-revs` gives us the average number of revisions of the two modules. To interpret the data, consider the `InfoUtils.java` module in the example output above. The coupling tells us that each time it's modified, it's a 78% risk/chance that we'll have to change our `Page.java` module too. Since there's probably no reason they should change together, the analysis points to a part of the code worth investigating as a potential target for a future refactoring.

### Visualizing the result

Future versions of Code Maat are likely to include direct visualization support. For now, I work with a set of standalone tools. I've open sourced one of those as [Metrics Tree Map](https://github.com/adamtornhill/MetricsTreeMap):

![coupling visualized](doc/imgs/tree_map_sample.png).

An alternative is to save the generated CSV to a file and import it into a spreadsheet program such as OpenOffice or Excel. That allows us to generate charts such as the ones below:

![coupling visualized](doc/imgs/coupling_sample.png).

## Code churn measures

Code churn is related to post-release defects. Modules with higher churn tend to have more defects. There are several different aspects of code churn. I intend to support several of them in Code Maat.

### Absolute churn

The absolute code churn numbers are calculated with the `-a abs-churn` option. Note that the option is only available for `git`. The analysis will output a CSV table with the churn accumulated per date:

             date,       added, deleted
             2013-08-09,   259,      20
             2013-08-19,   146,      77
             2013-08-21,     5,       6
             2013-08-20,   773,     121
             2013-08-30,   349,     185
             ...

Visualizing the result allows us to spot general trends over time:

![abs churn visualized](doc/imgs/abs_churn_sample.png).

### Intermediate results

Code Maat supports an `identity` analysis. By using this switch, Code Maat will output the intermediate parse result of the raw VCS file. This can be useful either as a debug aid or as input to other tools.

### JVM options

Code Maat uses the Incanter library. By default, Incanter will create an `awt frame`. You can surpress the frame by providing the following option to your `java` command: `-Djava.awt.headless=true`.
Code Maat is quite memory hungry, particularly when working with larger change sets. Thus, I recommend specifying a larger heap size than the `JVM` defaults: `-Xmx4g`.
Note that when running Code Maat through [leiningen](https://github.com/technomancy/leiningen), those options are already configured in the `project.clj` file.

## Limitations

The current version of Code Maat processes all its content in memory. Thus, it doesn't scale to large input files. The recommendation is to limit the input by specifying a sensible start date (as discussed initially, you want to do that anyway to avoid confounds in the analysis).

## Future directions

In future versions of Code Maat I plan to add more analysis methods such as code churn and developer patterns.
I also plan on direct visualization support and a database backed analysis to allow processing of larger log files. Further, I plan to add a worked example. That example will be a case study of some well-known open source code. Until then, I hope you find Code Maat useful in its initial shape.
