pathoscore
==========

pathoscore evaluates variant pathogenicity tools and scores.

evaluating scores is hard because logic can be circular and benign and pathogenic sets are
hard to curate and evaluate.

`pathoscore` is software and datasets that facilitate applying evaluating pathogenicity scores.

The sections below describe the tools

Annotate
--------

Annotate a vcf with some scores (which can be bed or vcf).
Note that this tool is a simple wrapper around [vcfanno](https://github.com/brentp/vcfanno) so 
a user can instead use to run vcfanno directly.

```
python pathoscore.py annotate \
    --scores exac-ccrs.bed.gz:exac_ccr:14:max \
    --scores mpc.regions.clean.sorted.bed.gz:mpc_regions:5:max \
    --exclude /data/gemini_install/data/gemini_data/ExAC.r0.3.sites.vep.tidy.vcf.gz \
    --conf combined-score.conf \
    testing-denovos.vcf.gz
```

The individual flags are described here:

### scores


The `scores` format is `path:name:column:op` where:

+ name becomes the new name in the INFO field.
+ column indicates the column number (or INFO name) to pull from the scores VCF.
+ op is a `vcfanno` operation.

### exclude

is a population VCF that is use to filter would-be pathogenic variants (as we know that common variants
can't be pathogenic). This can also be a set of regions to exclude.

### conf

an optional [vcfanno](https://github.com/brentp/vcfanno) conf file so users can specify exactly
how to annotate if they feel comfortable doing so.

This can also be used to specify vcfanno `[[postannotation]]` blocks, for example, to combine scores.

An example `conf` to combine 2 scores looks like:

```
[[postannotation]]
name="combined"
op="lua:exac_ccr+10\*cadd"
fields=["exac_ccr", "cadd"]
type="Float"
```

Evaluate
--------

```
python pathoscore.py evaluate \
    -s MPC \
    -s exac_ccr \
    -i mpc_regions \
    -s combined \
    pathogenic.vcf.gz \
    benign.vcf.gz
```

This will take the output(s) from `annotate` and create ROC curves and score distribution plots.
It assumes that the first VCF contains pathogenic variants and the 2nd contains benign variants.
It uses the columns specified via `-s` and `-i` as the scores.

`-i` indicates that lower scores are more constrained where as 

`-s` is for fields where higher scores are more constrained.

Output
------

An example ROC curve for the Clinvar truth-set looks like this:

![roc](https://user-images.githubusercontent.com/1739/29724634-6b730c44-8986-11e7-8b82-4341edcb3f0a.png "roc")

The point in the plot shows the max [J Statistic](https://en.wikipedia.org/wiki/Youden%27s_J_statistic) which can be
summarized as the point in each curve where the vertical distance to the Y=X line is maximized. 
This has its highest possible value at an FPR of 0 so there is an implicit penalty for having a high TPR at a high-ish
FPR.

We also report the full distrubtion of J statistics:


![J](https://user-images.githubusercontent.com/1739/29724633-6b72ee30-8986-11e7-9e1d-1033392e2914.png "J")

finally, we report the proportion of *benign* and *pathogenic* variants scored in a truth-set:


![scores](https://user-images.githubusercontent.com/1739/29724635-6b72f308-8986-11e7-89bb-3e86fc16fab7.png "scores")

These plots, along with the score-distributions for each method for pathogenic and benign, are aggregated into a single
HTML report.

Install
-------

Download a [vcfanno binary](https://github.com/brentp/vcfanno/releases) for your system and make it available as
`vcfanno` on your `$PATH`

Then run:
```
pip install -r requirements.txt
```

Then you should be able to run the evaluation scripts.

Truth Sets
----------

Part of `pathoscore` is to provide curated truth sets that can be used for evaluation.

These are kept in `truth-sets/`. Each set has a benign and/or a pathogenic set. 

Pull-requests for recipes that add new truth sets are welcomed. These should include a `make.sh`
script that, when run will pull from the original data source and make a benign and/or pathogenic
vcf that is bgzipped and tabixed and made as small as possible (see the clinvar example for how
to remove unneeded fields from the INFO field).

All truth-sets should be annotated with `bcftools csq` so that it's possible to choose to score only
functional variants.

Currently we have:

### clinvar

+ clinvar pathogenics are either `Pathogenic` or `Likely-Pathogenic` and variants with uncertainty are removed.
+ clinvar benigns are either `Benign` or `Likely-Benign` and variants with uncertainty are removed.
+ clinvar variants where there is an SSR field are removed because they are suspected false positives due to paralogy or computational/sequencing error

### samocha

These are from [Kaitlin Samocha's paper](http://www.biorxiv.org/content/early/2017/06/12/148353) on mis-sense contraint.

+ benigns are labelled as `control` in her source file
+ pathogenics are anything other than control.


