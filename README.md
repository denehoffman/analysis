<p align="center">
  <h1 align="center">analysis</h1>
</p>
<p align="center">
    <h3 align="center">A tool to manage CMU GlueX analyses with SLURM and Python</h3>
</p>

<p align="center">
  <a href="https://www.cmu.edu/physics/"><img alt="CMU Physics Homepage" src="https://img.shields.io/badge/CMU-C41230"></a>
  <a href="https://halldweb.jlab.org/wiki/index.php/Main_Page"><img alt="GlueX Wiki" src="https://img.shields.io/badge/Gluex-33348e"></a>
  <a href="https://github.com/denehoffman/analysis/commits/main/"><img alt="GitHub Last Commit" src="https://img.shields.io/github/last-commit/denehoffman/analysis/main"></a>
  <img alt="GitHub lines of Code" src="https://img.shields.io/tokei/lines/github/denehoffman/analysis">
  <img alt="GitHub Top Language" src="https://img.shields.io/github/languages/top/denehoffman/analysis">
  <a href="https://github.com/denehoffman/analysis/blob/main/LICENSE"><img alt="GitHub License" src="https://img.shields.io/github/license/denehoffman/analysis"></a>
</p>

I built this project because I realize I will soon be embarking on the systematic-error-identifying part of my research, and will therefore have to run my analysis pipeline more often. In search of efficiency, organization, and ease of use, I've settled on this tool. All analyses will be output to generally the same location in a nice and neat way, extensive logs are kept for each part of the pipeline, and SLURM jobs are made dependent on each other, so all jobs will run in sequence and you are free to use your terminal emulator again.

The basic structure is as follows: You specify some set of `analyses`, which are just groups of ROOT files you want to pass through your DSelector, then you create a single `study` which will contain the same info for each `analysis` and maybe run some `extensions` on the output files. For example, analyses could be `RunPeriod`s, so maybe all of the data for Spring 2017 is located in a folder pointed to by the `S17` analysis. Then you could run a `study` over `S17`, `S18`, `F18`, and `S20` (or whatever your chosen names might be) all together to run the entire GlueX dataset (at time of writing). You could also create groups which represent multiple analyses in a single tag, like `data` for all of the data and `mc` for the accepted Monte-Carlo. Then you could run both together using `--group data,mc`!

## Usage

General usage is described in the `--help` text:
```sh
$ analysis --help
Usage:
    analysis
    analysis (run | r) [options] (<analysis_names>... | --group <group_names>) [--name <study_name>]
    analysis (list | ls | l) [--group <group_names>]
    analysis -h | --help

Options:
    -c, --clean                remove individual run files after merging. [default: False]
    -d, --description <text>   enter the description in command line (open $EDITOR otherwise).
    -e, --ext <extensions>     use extensions specified in ~/.analysis.toml (comma-separated).
    -f, --force                force execution of analysis and overwrite existing analysis.
    -g, --group <group_names>  select by group, names may be comma-separated list.
    -n, --name <study_name>    name of study.
    -q, --queue <queue_name>   specify queue by name (eg: red). Defaults to queue with fewest
                               queued jobs with ties broken by config file listing order.
    -u, --use <n>              use only the first <n> files in each analysis (for testing)
    -h, --help                 show this help message and exit.

Commands:
    <none>                     echo path to output files
    run [alias: r]             run analysis over specified analyses.
    list [aliases: ls, l]      list valid analysis identifiers.
```

Beyond this, the user also needs to create a file called `.analysis.toml` in their home directory (`~/.analysis.toml`, note the dot in front of the filename), or they should point the environment variable `$ANALYSIS_CONFIG` to the location of a `TOML` file by a custom path. If neither exist, the program will generate some boilerplate to show the typical contents:

```toml
[general]
output-directory = "/<raid>/<user>/analysis/"  # Replace with your path
log-directory = "/<raid>/<user>/analysis/logs" # Replace with your path
env-file = "/home/<user>/setup.sh"             # Replace with your path

queues = [
  { name = "blue", total-memory = 128000, cpu-threads = 2, total-nodes = 16, cpus-per-node = 32, node-share = 1.0 },
  { name = "green", total-memory = 64000, cpu-threads = 1, total-nodes = 8, cpus-per-node = 32, node-share = 1.0 },
  #{ name = "red", total-memory = 64000, cpu-threads = 2, total-nodes = 11, cpus-per-node=32, node-share = 1.0 },   # red is temporarily offline
]

[analysis]

[analysis.S17]
input-directory = "/path/to/spring_17/merged"
version = "ver52"
dselector = "/path/to/DSelector.C"
group = "data"                                 # optional

[extension]

[extension.polarize]
command = "/home/nhoffman/bin/polarize"               # command takes arguments and creates a new file <filename>_polarized.root
arguments = ["@flattree.root", "-t", "polarized"]     # valid tags are @flattree, @tree, and @hist
output = { "@flattree" = "@flattree_polarized" }      # give mappings for the output files if you want to create a pipeline of extensions
```

Fill in the paths specified. You should have some kind of `setup.sh` file which you typically source to set all of the analysis-related environment variables. For `queues`, you probably don't need much modification. The `node-share` is the fraction of the total nodes you want to use, and values other than `1.0` are what I would classify as experimental (and slow).

Next, there's a section for analyses. To add a new one, simply create a new table element:
```toml
[analysis.<analysis_name>]
input-directory = "enter the full path to a directory containing all your .root files before the DSelector"
version = "this is where the analysis version tag goes, don't want to be mixing those...usually ver##"
dselector = "full path to your DSelector_blah_blah_blah.C file"
group = "this is optional, but will allow you to run multiple analyses if they all belong to the same group"
```

The other main part of the config is the extensions. These are just little scripts that run over the output files from your analysis and produce extra files. They can also modify the filestem of an output file and pass it on to the next extension, chaining them in order. For instance, you could create two extensions for which running `analysis run S17 -e ext1,ext2 --name test_a` and `analysis run S17 -e ext2,ext1 --name test_b` will yield different results, since `test_a` had `ext1` run first and the resulting files were passed to `ext2`, and `test_b` had the opposite order. This is accomplished by input and output tags, which are replaced when the script is run. There are three tags, `@tree`, `@hist`, and `@flattree`, corresponding to the three kinds of merged files output from the analysis through the DSelector. The extension in the example is called `polarize`, and it runs a script called `polarize` which will take the existing `flattree_<study_name>.root` and generate a new file `flattree_<study_name>_polarized.root`. We could then make another extension, say the following,
```toml
[extension.splot]
command = "/home/nhoffman/bin/splot"
arguments = ["@flattree.root"]
output = { "@flattree" = "@flattree_splot" }
```
which could then be run in the same command, and would take `flattree_<study_name>_polarized.root` and output a new file `flattree_<study_name>_polarized_splot.root`. Note that in the previous extension, there was a `-t <tag>` option that tells the `polarize` command what suffix to add to the file, so we had to specify that it adds `_polarized` in two different places whereas we only needed to specify it in the `output` here.

Finally, you can add your own keys to the next iterations to create custom pipelines outside of the three tagged files. For instance, the `splot` command I use actually creates a PDF plot at `flattree_<study_name>_splot.pdf`. We don't actually have to tell `analysis` about it in the config file, but if we want some later extension to do something with it, we could change the `output` to the following:

```toml
output = { "@flattree" = "@flattree_splot" , "@splotplot" = "@flattree_splot"}
```

I'll admit, this does look a bit silly and I'm working on a way to make it more clear what is happening. We could create another program which tages arguments like:
```toml
[extension.extract]
command = "/magic/command/to/extract/a/page" # Takes a PDF file and a page number as input and outputs <path>_<page_number>.pdf
arguments = ["@splotplot.pdf", "-p", "3"] # get the third page
output = { "@splotplot" = "@splotplot_3" }
```

When this extension is run, another extension can still refer to the `@flattree` tag in arguments without having to deal with the `_3` suffix which was added to the PDF. If you're not sure what all this accomplishes, don't worry, you'll probably never use it.

## Running

The basic commands are `list` and `run`. As their names suggest, `list` lists information about available analyses, so you can refer to it without having to open up your config all the time. You can even list just the information for a group of analyses using the `--group` argument here.

The main command is `run`, which actually submits the analysis jobs to the SLURM queue. We can provide it with either a space-separated list of analyses, or with the `--group` argument. It is *very important* to note that you *must* supply a name for the study using the `--name <study_name>` argument.

The script will then run several `sbatch` commands. The first will dispatch the individual root files to their respective nodes, process them through your DSelector, and copy back the results. Note that I have hardcoded a requirement to copy back tree, histogram, and flattree files. If your DSelector doesn't ouput all of those, it should still run fine, but you might get some errors in some of your scripts, although I believe they will not impede the actual execution. The second command waits for all of the jobs to finish, and then it runs an `hadd` over the respective sets of files. Finally, more jobs will be spawned, dependent on this `hadd` job, to run the various extensions specified by the `--ext <ext1>,<ext2>,...` argument. These extension jobs will each be dependent on the success of the previous so that they always execute in order.

Note that I have not yet written code to rerun just the jobs that failed, or code which recognizes jobs have failed and stops or restarts the pipeline. This is in the works, so for now you'll just have to hope nothing terrible goes wrong and read your log files.

If you have a study that you want to re-run, overwriting the contents of the folder, you should use the `--force` argument. This is typically used if some part of your pipeline failed and you need to reset everything but want to use the same study name without manually deleting the file. This, like the `--use` command, which only runs a few files through the DSelector, are mostly used for testing.

Finally, unless you specify some text via the `--description <text>` argument, the program will open up a text editor (whatever you have bound to `$EDITOR` in your shell, or `vim` if it's unbound) and will prompt you to add a description for this study, much like a `git commit` message. These descriptions will be available later in a `desc` plaintext file inside the study directory. Saving and exiting will end the program, although a sneaky `Ctrl+C` will probably exit without creating a `desc` file, if you're that desperate to forget why you're running the study. Remember, to save and exit out of `vim`, hit `Esc` a few times for good measure to enter `Normal` mode, then type `:wq` and hit `Enter`.

Here are some examples of some hypothetical run commands
```bash
$ analysis run S17 S18 F18 -n phase_1 -d "Made some changes to the analysis today..."
# run over Phase I data

$ analysis run --group accmc -n monte_carlo_new_cut -e polarize,coherent_peak,splot
# run over some group of analyses containing Monte-Carlo, and run some extensions to polarize, get the coherent peak, then sPlot

$ analysis run --group data -n data_new_cut --queue green
# run analysis over the data group on the green queue
```

## Conclusion

That should be all you need to start using this program. If you really need to modify things on a finer detail, like changing the max time for some of the `sbatch` scripts inside the source code, I recommend you fork this repository rather than make edits inside the main repo itself, this will save a lot of time if you have a new feature you'd like to add.

Additionally, given the way extensions work, its in your best interest to create some tools which make various plots or selections that can't be done in the DSelector. For instance, running sPlot is great, but it would be nice to run some additional scripts on the weighted result to be able to see some mass spectrum at a glance, for instance. Or you could make an extension script which selects the best combo from each event in a tree or flattree!

A final note, I'm hesitant to add the `group` structure to extensions. Although you might have a long list of programs which need to be run, I would rather you just put stuff into a command alias, or even write a [Justfile](https://github.com/casey/just) to handle your entire analysis. I personally will be using my other project, [PSelector](https://github.com/denehoffman/PSelector), to automatically generate DSelectors from another `TOML` file and a `Justfile` to update the current DSelector according to that file and run a study over my data.
