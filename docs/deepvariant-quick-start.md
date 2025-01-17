# DeepVariant quick start

This is an explanation of how to use DeepVariant.

## Background

To get started, you'll need the DeepVariant programs (and some packages they
depend on), some test data, and of course a place to run them.

We've provided a Docker image, and some test data in a bucket on Google Cloud
Storage. The instructions below show how to download the data through the
corresponding public URLs from these data.

### Update in r0.8 : Use Docker to run DeepVariant in one command.

In the 0.8 release, we are introducing one convenient command that will run
through all 3 steps that are required to go from a BAM file to the VCF/gVCF
output files. You can still read about the r0.7 approach in
[Quick Start in r0.7].

If you want to compile the DeepVariant binaries for yourself, we also have a
[Dockerfile] that you can use to build your own Docker image. You can read the
[docker build] documentation on how to build.

For production use cases on larger scale of dataset, we recommend looking into
the [External Solutions] section.

## Get Docker image, models, and test data

### Get Docker image

```bash
BIN_VERSION="0.8.0"

sudo apt -y update
sudo apt-get -y install docker.io
sudo docker pull gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}"
```

### Download test data

Before you start running, you need to have the following input files:

1.  A reference genome in [FASTA] format and its corresponding index file
    (.fai).

1.  An aligned reads file in [BAM] format and its corresponding index file
    (.bai). You get this by aligning the reads from a sequencing instrument,
    using an aligner like [BWA] for example.

We've prepared a small test data bundle for use in this quick start guide that
can be downloaded to your instance from the public URLs.

Download the test bundle:

```bash
INPUT_DIR="${PWD}/quickstart-testdata"
DATA_HTTP_DIR="https://storage.googleapis.com/deepvariant/quickstart-testdata"

mkdir -p ${INPUT_DIR}
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/NA12878_S1.chr20.10_10p1mb.bam
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/NA12878_S1.chr20.10_10p1mb.bam.bai
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/test_nist.b37_chr20_100kbp_at_10mb.bed
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/test_nist.b37_chr20_100kbp_at_10mb.vcf.gz
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/test_nist.b37_chr20_100kbp_at_10mb.vcf.gz.tbi
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/ucsc.hg19.chr20.unittest.fasta
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/ucsc.hg19.chr20.unittest.fasta.fai
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/ucsc.hg19.chr20.unittest.fasta.gz
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/ucsc.hg19.chr20.unittest.fasta.gz.fai
wget -P ${INPUT_DIR} "${DATA_HTTP_DIR}"/ucsc.hg19.chr20.unittest.fasta.gz.gzi
```

This should create a subdirectory in the current directory containing the actual
data files:

```bash
ls -1 ${INPUT_DIR}
```

outputting:

```
NA12878_S1.chr20.10_10p1mb.bam
NA12878_S1.chr20.10_10p1mb.bam.bai
test_nist.b37_chr20_100kbp_at_10mb.bed
test_nist.b37_chr20_100kbp_at_10mb.vcf.gz
test_nist.b37_chr20_100kbp_at_10mb.vcf.gz.tbi
ucsc.hg19.chr20.unittest.fasta
ucsc.hg19.chr20.unittest.fasta.fai
ucsc.hg19.chr20.unittest.fasta.gz
ucsc.hg19.chr20.unittest.fasta.gz.fai
ucsc.hg19.chr20.unittest.fasta.gz.gzi
```

### Model location (optional)

Starting from r0.8, we put the model files inside the released Docker images.
So there is no need to download model files anymore. If you want to find the
model files of all releases, you can find them in our bucket on the Google Cloud
Storage. You can view them in the browser:
https://console.cloud.google.com/storage/browser/deepvariant/models/DeepVariant

## Run DeepVariant with one command

DeepVariant consists of 3 main binaries: `make_examples`, `call_variants`, and
`postprocess_variants`. To make it easier to run, we create one entrypoint that
can be directly run as a docker command. If you want to see the details, you can
read through [docker_entrypoint.py].

```bash
OUTPUT_DIR="${PWD}/quickstart-output"
mkdir -p "${OUTPUT_DIR}"
```

Using the Docker entrypoint ([docker_entrypoint.py]), you can run everything
with the following command:

```bash
sudo docker run \
  -v "${INPUT_DIR}":"/input" \
  -v "${OUTPUT_DIR}:/output" \
  gcr.io/deepvariant-docker/deepvariant:"${BIN_VERSION}" \
  /opt/deepvariant/bin/run_deepvariant \
  --model_type=WGS \
  --ref=/input/ucsc.hg19.chr20.unittest.fasta \
  --reads=/input/NA12878_S1.chr20.10_10p1mb.bam \
  --regions "chr20:10,000,000-10,010,000" \
  --output_vcf=/output/output.vcf.gz \
  --output_gvcf=/output/output.g.vcf.gz
```

This will generate 4 files in `${OUTPUT_DIR}`:

```bash
ls -1 ${OUTPUT_DIR}
```

outputting:

```
output.g.vcf.gz
output.g.vcf.gz.tbi
output.vcf.gz
output.vcf.gz.tbi
```

## Evaluating the results

Here we use the `hap.py`
([https://github.com/Illumina/hap.py](https://github.com/Illumina/hap.py))
program from Illumina to evaluate the resulting 10 kilobase vcf file. This
serves as a quick check to ensure the three DeepVariant commands ran correctly.

```bash
sudo docker pull pkrusche/hap.py
sudo docker run -it \
  -v "${INPUT_DIR}":"/input" \
  -v "${OUTPUT_DIR}:/output" \
  pkrusche/hap.py /opt/hap.py/bin/hap.py \
  /input/test_nist.b37_chr20_100kbp_at_10mb.vcf.gz \
  /output/output.vcf.gz \
  -f "/input/test_nist.b37_chr20_100kbp_at_10mb.bed" \
  -r "/input/ucsc.hg19.chr20.unittest.fasta" \
  -o "/output/happy.output" \
  --engine=vcfeval \
  -l chr20:10000000-10010000
```

You should see output similar to the following.

```
Benchmarking Summary:
  Type Filter  TRUTH.TOTAL  TRUTH.TP  TRUTH.FN  QUERY.TOTAL  QUERY.FP  QUERY.UNK  FP.gt  METRIC.Recall  METRIC.Precision  METRIC.Frac_NA  METRIC.F1_Score  TRUTH.TOTAL.TiTv_ratio  QUERY.TOTAL.TiTv_ratio  TRUTH.TOTAL.het_hom_ratio  QUERY.TOTAL.het_hom_ratio
 INDEL    ALL            4         4         0           13         0          9      0       1.000000                 1        0.692308         1.000000                     NaN                     NaN                   0.333333                   1.000000
 INDEL   PASS            4         4         0           13         0          9      0       1.000000                 1        0.692308         1.000000                     NaN                     NaN                   0.333333                   1.000000
   SNP    ALL           44        43         1           59         0         16      0       0.977273                 1        0.271186         0.988506                     1.2                    1.36                   0.333333                   0.340909
   SNP   PASS           44        43         1           59         0         16      0       0.977273                 1        0.271186         0.988506                     1.2                    1.36                   0.333333                   0.340909
```

[BAM]: http://genome.sph.umich.edu/wiki/BAM
[BWA]: https://academic.oup.com/bioinformatics/article/25/14/1754/225615/Fast-and-accurate-short-read-alignment-with
[docker build]: https://docs.docker.com/engine/reference/commandline/build/
[docker_entrypoint.py]: https://github.com/google/deepvariant/blob/r0.8/scripts/docker_entrypoint.py
[Dockerfile]: https://github.com/google/deepvariant/blob/r0.8/Dockerfile
[External Solutions]: https://github.com/google/deepvariant#external-solutions
[FASTA]: https://en.wikipedia.org/wiki/FASTA_format
[Quick Start in r0.7]: https://github.com/google/deepvariant/blob/r0.7/docs/deepvariant-quick-start.md
[VCF]: https://samtools.github.io/hts-specs/VCFv4.3.pdf
