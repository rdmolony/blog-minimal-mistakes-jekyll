---
title: "Codema-dev Projects"
category: blog
tags:
  - link
  - prefect
  - Python
  - docker
  - Jupyter Notebook
  - pandas
  - requests
---
A collection of Extract Transform Load (ETL) projects used in the generation of various maps on the `codema-dev.github.io` website.  All maps on `codema-dev.github.io` are generated by either an ad-hoc `py` notebook saved in `deprecated` or in a folder on `projects`.

The creation of a bottom-up Dublin energy model requires reading data from various sources (**Extract**), cleaning and merging it on common values (**Transform**) and finally loading it to a file or database (**Load**).  Implementation of the above process has gone through numerous phases.

Initially all ETL was run in a single `Jupyter Notebook` using `pandas` to wrangle data and `requests` to download it.  It grew so rapidly that not even section headers could make it readable.  In my efforts to clean this up I found [Vlad Kazantsev's video on "Clean Code in Jupyter notebooks"](https://www.youtube.com/watch?v=2QLgf2YLlus).  Vlad advises using smaller notebooks that "do one thing" so I split my mega-notebook into multiple smaller notebooks.  Now the challenge was to "glue" these together.  I decided to number the notebooks and document the run-order. Problems remained.  The notebooks didn't work well with `git` so it was tricky to inspect changes.  I kept hearing that unit testing is essential but didn't know how to unit test my notebooks.  [Joel Gru's video "I don't like notebooks"](https://www.youtube.com/watch?v=7jiPeIFXb6U) pushed me towards writing `Python` functions in `vscode` instead.  I also used `docker` was used to create a sharable environment with the caveat that users on the other end must have basic familiarity with the command line.

I decided to use `prefect` to glue them together and created [drem](https://github.com/codema-dev/drem).  I chose `prefect` as it was easy to get started with.  Now it was possible to unit test my functions (as in [Ned Batchedler's talk](https://www.youtube.com/watch?v=FxSsnHeWQBY)) to make sure they were doing what I thought they were doing and to run test pipelines on sample data (as in [Nick Radcliffe's talk](https://www.youtube.com/watch?v=fw6P6VFPo24)) to ensure they were alive and well.  This switch didn't come without problems.  In time the pipeline grew complex as more data sources were added to it.  It also didn't gel well with ad-hoc visualisations as running a single task requires re-running the entire pipeline.  I tried [checkpointing](https://docs.prefect.io/core/concepts/persistence.html#output-caching-based-on-a-file-target) to cache the output of tasks and so their skip execution, however, it still proved easier to create ad-hoc notebooks for data exploration tasks than to split out the logic into multiple functions.  Since these notebooks lacked "glue" sharing them was tricky.

I eventually returned to `prefect` in [`projects`](https://github.com/codema-dev/projects)  to glue tasks together once again using the `prefect` `jupyter` extension.  I used `conda` instead of `docker` to share environments as it is a bit less involved.  I also created a pipeline per mini-project instead of maintaining a massive pipeline since most datasets update only infrequently.  If one pipeline requires the output of another pipeline it's straightforward to store outputs on `s3` and in this way share it across pipelines.  One issue that remains is use of closed-access data which will continue to hinder full reproducibility of the maps.

<div><a href="https://github.com/codema-dev/projects" class="btn btn--primary">Github</a></div>