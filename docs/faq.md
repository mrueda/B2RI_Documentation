# Frequently Asked Questions

## Data ingestion tools

### Are Beacon v2 `genomicVariations.variation.location.interval.{start,end}` coordinates 0-based or 1-based?

They are [0-based](http://docs.genomebeacons.org/formats-standards/#genome-coordinates)

### I have an error when attempting to use `beacon vcf`, what should I do?

* In 9 out 10 cases, the error comes from **BCFtools** and is about the **reference genome** used. The reference genome is set up inside the **parameters** [file](https://github.com/mrueda/beacon2-ri-tools) and the possibilities are _hg19_, _hg38_ (both use `chr` before the number), and _hs37_ (does not use `chr` before the number). Be aware that BCFtools is very nit picky (with a reason) about the contigs, etc. not matching those in the fasta file. Please fix your VCF accordingly or modify `config.yaml` to provide the path to your reference genome.

* On top of that, **BCFtools** may complain about the number of fields somewhere (e.g., at INFO) not being right.
```
INFO field IDREP only contains 1 field, expecting 2
```
You can try solving this issue manually or use `bcftools annotate` to get rid of the problematic fields, like this:

    bcftools annotate -x INFO/IDREP input.vcf.gz | gzip > output.vcf.gz

### Can I use SINGLE-SAMPLE and MULTI-SAMPLE VCFs?

Yes, you can use both. MongoDB allows for incremental loads so if you have single sample VCFs that's ok too (you don't need to merge them into a multisample VCF). The connection between sample and variants is performed at the `datasets` collection (and/or `cohorts`).

### Can I use genomic VCF ([gVCF)](https://gatk.broadinstitute.org/hc/en-us/articles/360035531812-GVCF-Genomic-Variant-Call-Format)?

Yes, but **first you will need to transform them** to a VCF. There are quite a few sophisticated ways to do this (e.g., `bcftools convert --gvcf2vcf --fasta ref.fa input.g.vcf`). Here, we are not interested in having the whole genome positions, but rather the positions that contain ALT alleles. A "quick and dirty" solution to get those can be achieved with common Linux tools:

    zcat input.g.vcf.gz | awk '$5 != "<NON_REF>"' | sed 's#,<NON_REF>##' | gzip > output.vcf.gz 

### Why are we re-annotating VCFs | Can I use my own annotations?

The underlying idea about annotating with the B2RI's data ingestion tools is to provide consistency/homogeneicity for the community. Technically speaking, to create `genomicVariationsVcf.json.gz` BFF, first we need to parse an annotated VCF. It's hard to create a parser that handles all the annotation alternatives from the community. On top of that, we need to make sure that the _essential_ fields exist. That's why recommend annotating (or re-annotating if your VCFs already have annotations) VCFs with our tooling. Note that previous annotations in the VCF will be discarded.  

Having said that, in ocassions researchers have **internal annotations** that can have a lot of value as well. For that reason, it is also possible to add alternative genomic variations by filling out the corresponding _tab_ in the provided [XLSX](https://github.com/mrueda/beacon2-ri-tools/blob/main/utils/bff_validator/Beacon-v2-Models_template.xlsx). Just make sure you fill out all the mandatory terms requested in the schema. The resulting file will be named `genomicVariations.json`. Both `genomicVariations.json` and `genomicVariationsVcf.json.gz` (see above paragraph) will end up loaded in the MongoDB collection _genomicVariations_. See more information in this [tutorial](./tutorial-data-beaconization.md).

### Is there an alternative to the Excel file for generating metadata/phenotypic data?

Yes, there is. You can use CSV or JSON files directly as input for the `bff-validator` utility. For detailed instructions, please refer to the [bff-validator manual](https://github.com/mrueda/beacon2-ri-tools/tree/main/utils/bff_validator).

Alternatively, if your clinical data is in REDCap, OMOP CDM, Phenopackets v2, or simply in a raw CSV format, we recommend using the [Convert-Pheno](https://github.com/CNAG-Biomedical-Informatics/convert-pheno) tool. Convert-Pheno is tailored towards the `individuals` entity of the Beacon v2 Models.

### `bff-validator` specification mismatches

By default, `bff-validator` will validate your data against the default schemas installed with your `beacon2-ri-tools` version. In this regard, it can happen that `bff-validator` gives you warnings on things that look OK [elsewhere](https://github.com/ga4gh-beacon/beacon-v2/issues). An example of this could be warnings on objects matching more than one possibility in `oneOf` keywords. If this happens just use the flag `--ignore-validation` once you are ready to create your `.json` files. 

### Do you load all variations present in a VCF file?

Yes, we do not apply any filter such as using `FILTER` or `QUAL` fields (but we do store those values in case they need to be used _a posteriori_).

### Do you have any recommendations in how to speed up the data ingestion process?

Usually, metadata/phenoclinic data ingestion is fast as we'll be dealing with thousands of values (or dozens of thousands). Processing this should not take more than seconds/minutes.

**VCF processing** is what takes more time, in particular if you have **WGS** (e.g., you can have > 100M variants and thousands of samples). These are some recommendations:

1. Split your VCF by chromosome.

    For this you can use community-tools:
 
        bcftools view input.vcf.gz --regions chr1

    or
    
        tabix -p vcf input.vcf.gz
        tabix input.vcf.gz chr1 | bgzip > chr1.vcf.gz

    or Linux tools (see more [examples](https://bioinformatics.stackexchange.com/questions/3401/how-to-subset-a-vcf-by-chromosome-and-keep-the-header)):

        zcat input.vcf.gz | awk '/^#/ || $1=="chr1"' | bgzip > chr1.vcf.gz

2. Use [parallel processing](https://github.com/mrueda/beacon2-ri-tools/tree/main/utils/bff_queue) to submit the jobs. 

### Can I use parallel jobs to perform data ingestion into mongoDB?

Yes, yet this may slow down a bit the ingestion itself.

### When performing incremental uploads, do I need to re-index MongoDB?

Nope. The indexes are created during the first load of key/values and updated automatically on every insert operation. Next attempts for re-indexing are simply [discarded by MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.createIndex/?_ga=2.111853831.262597912.1652622239-541609842.1652622239) (i.e., the operation is **idempotent**).

### Where do I get the WGS VCF for the he CINECA synthetic cohort EUROPE UK1?

You can download [chr22](https://github.com/mrueda/beacon2-ri-tools/tree/main/CINECA_synthetic_cohort_EUROPE_UK1#external-files-crg-public-ftp-site) from our FTP site. If you need the WGS, you must request access and download it from the [EGA](https://ega-archive.org/datasets/EGAD00001006673). The VCF file contains WGS data for 2,504 synthetic individuals (~20 GB). For more information, see [this document](./synthetic-dataset.md).

## Beacon v2 API 

### Which tool do you recommend for making queries?

We recommend [Curl](https://curl.se).

### Why are the API queries slow?

Ensure that you create an index for **single fields** and **text fields** in MongoDB. Without proper indexing, query performance will be significantly slower.

### Is the response data encrypted?

By default, the API will start as `http`. The request/response can be encripted by adding the `https` protocol on top.

## B2RI (General)

### Is B2RI free?

**Yes**, it is open sourc and it is free. The data ingestion tools have [GNU General Public License v3.0](https://en.wikipedia.org/wiki/GNU_General_Public_License#Version_3) and the API has [Apache License v2.0](https://www.apache.org/licenses/LICENSE-2.0). 

The included [CINECA_synthetic_cohort_EUROPE_UK1](https://www.cineca-project.eu/cineca-synthetic-datasets) dataset has [CC-BY](https://en.wikipedia.org/wiki/Creative_Commons_licens) license.

### I am using a SQL-based database, can I still use your Reference Implementation?

The issue with using a SQL-based database is that, if you want to be **Beacon v2 response _compliant_**, you will need to convert your tables-based data to the JSON Schema of the [Beacon v2 Models](http://docs.genomebeacons.org). Intrepid implementers can perform this transformation at the API level. However, a much simpler alternative (and one commonly seen in healthcare systems) is to perform a `dump` (data export) of the subset of data you want to share and then use [beacon2-ri-tools](tutorial-data-beaconization.md) to convert this tabular data to Beacon v2 format and use the included REST API. So yes, if you follow this path you will still be using our Reference Implementation.

Additionally, OMOP CDM instances in PostgreSQL can be converted to BFF data exchange files using the [Convert-Pheno](https://github.com/CNAG-Biomedical-Informatics/convert-pheno) tool, which is tailored towards the `individuals` entity of the Beacon v2 Models.

### Should I update to the `latest` version?

Yep. We recommend checking our Github repositories ([beacon2-ri-tools](https://github.com/mrueda/beacon2-ri-tools) and [beacon2-ri-api](https://github.com/EGA-archive/beacon2-ri-api)) and downloading the latest version.

### How do I cite B2RI?

!!! Note "Citation"
    Rueda, M, Ariosa R. "Beacon v2 Reference Implementation: a toolkit to enable federated sharing of genomic and phenotypic data". _Bioinformatics_, btac568, [DOI](https://doi.org/10.1093/bioinformatics/btac568).
