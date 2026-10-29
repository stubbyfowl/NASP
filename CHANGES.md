# Changes on `fix/modernize-deps`

Small fixes that let NASP run on current Python and current samtools. No change to
NASP's results, no Go changes, no version bump. Background: the 2026 diagnosis
of the broken bioconda package (FINDINGS.md, kept outside this repo).

## 1. `pkg_resources` → `importlib.resources`

setuptools removed `pkg_resources` in 82.0.0. NASP used it to find files shipped inside the
package, so on a current environment these crash with
`ModuleNotFoundError: No module named 'pkg_resources'`:

- `nasp matrix` / `export` / `duplicates` / `frankenfasta`, and the matrix step of every run
  (`nasp/nasp.py`, `gonasp_path`)
- the default adapter file offered when Trimmomatic is enabled (`nasp/nasp.py`,
  `_get_trimming_parameters`)
- external-genome import (`nasp/convert_external_genome.py`)

All four call sites now use `importlib.resources.files()` from the standard library. That
needs Python 3.9+, so `setup.py` now declares `python_requires='>=3.9'`.

## 2. `samtools sort` syntax

Every aligner pipes its output through `samtools sort - <prefix>`. samtools 1.3 removed that
form, which is why NASP has been stuck on samtools 1.2 or older. The four call sites in
`nasp/dispatcher.py` now emit:

    samtools sort -T <prefix> -o <prefix>.bam -

The `-T` matters. Without it, samtools 1.1 writes no BAM and still exits 0. With it, the same
command works on samtools 1.1, 1.9 and 1.24.

## 3. Raw strings for regex literals

Eight regex patterns used escapes like `\.` and `\d` in plain strings. That gives a
`SyntaxWarning` on Python 3.12+ and will become an error in a future Python. They are now
`r'...'` strings. Each pattern was checked to be byte-for-byte identical before and after.

## 4. SAMtools/BCFtools SNP caller retired

This caller runs `samtools mpileup -uD | bcftools view -ceg`, which is samtools/bcftools 0.1.x
syntax. Every bcftools on bioconda rejects it (`Could not parse --min-ac eg`), so it could only
produce empty VCFs. It's now retired the same way SolSNP was:
- it's no longer offered in the interactive prompts, which now list GATK and VarScan;
- a config file that still names it is skipped with a clear message.

`_run_samtools` is left in place for anyone who later ports it to `bcftools call`.

## How it was tested

- **Full pipeline on `examples/example_1`** (BWA-mem + GATK 3.8 UnifiedGenotyper + external
  genome + duplicate check, Linux x86_64, Python 3.12). Three runs:
  - original code + samtools 1.1
  - this branch + samtools 1.1
  - this branch + samtools 1.24, started with `--config`

  All three gave byte-identical `master.tsv`, `bestsnp.tsv` and `missingdata.tsv`, and all three
  match `examples/example_1_output_files/` exactly (3,977 positions, the same 5 SNPs). The
  fixes change no results, and this branch works on current samtools.
- **Install and entry points** on Python 3.9, 3.10, 3.13 and 3.14, in environments with no
  `pkg_resources`. All five console scripts work, `nasp matrix` and `nasp export` reach the Go
  binary, and the bundled adapter FASTA is found.
- **`compileall` with `-W error::SyntaxWarning`** is clean on the same versions.

## Not changed (deliberately)

- **Go code.** `command/align`, `command/index` and `command/snpcall` don't compile and aren't
  used by `main.go`. `command/align/cmd.go` also still contains the old `samtools sort`
  form. The Go tests don't compile either. None of this affects the `nasptool` binary.
- **`setup.py` version** (still `1.2.0` at tag `v1.2.1`) and the Python 3.3/3.4 classifiers.
  Those are the maintainer's call.

## For the bioconda recipe

With these fixes in a release, the recipe can drop the `samtools >=1.1,<1.2` and
`setuptools <82` pins and build for every Python bioconda supports. Until then, the
recipe-only fix for 1.2.1 is limited to Python 3.10/3.11, because samtools 1.1 requires
OpenSSL 1.1.1 and conda-forge's Python 3.12+ requires OpenSSL 3.
